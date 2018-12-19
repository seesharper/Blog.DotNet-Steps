

I have been using C# scripting as a tool for writing build scripts for many years now. At first it was [ScriptCs](http://scriptcs.net/) using [Sublime Text](https://www.sublimetext.com/) as the editor. There was no intellisense or debugging capabilities, but still it was insanely powerful to have all the sweetness of C# available in a simple script file. 

This post is not really about the evolution of C# scripting, but rather about how I went about creating [dotnet-steps](https://github.com/seesharper/dotnet-steps) which is a super simple way of composing "steps" in a C# script. 

> "A small step for man kind, but a HUGE leap for build scripts"

For a build script it is pretty common to have various steps such as *build*, *test*, *pack*, *deploy* and so forth. These steps can then be composed together to form the flow of the build script and we can also execute the script in such a way that we can cherrypick the step(s) to be executed.

There are plenty of tools that provides similar concepts. 

 * Cake - Full fledged build system with all batteries included. 
 * Bullseye - Clean and simple "targets" runner written as a console application.
 * Nuke - Build scripts written as console application

These tools all have their own sort of DSL to declare and chain composable chucks of code together.

### Cake

```c#
Task("Run-Unit-Tests")
    .IsDependentOn("Build")
    .Does(() =>
{
    NUnit("./src/**/bin/" + configuration + "/*.Tests.dll");
});
```

### Bullseye

```C#
Target("default", DependsOn("drink-tea", "walk-dog"));
Target("make-tea", () => Console.WriteLine("Tea made."));
Target("drink-tea", DependsOn("make-tea"), () => Console.WriteLine("Ahh... lovely!"));
Target("walk-dog", () => Console.WriteLine("Walkies!"));
```

### Nuke

```c#
Target Compile => _ => _
    .Executes(() => { });

Target Pack => _ => _
    .DependsOn(Compile)
    .Executes(() => { });

Target Test => _ => _
    .DependsOn(Compile)
    .Executes(() => { });

Target Full => _ => _
    .DependsOn(Pack, Test);
```

My personal favourite among these three is the Nuke syntax. There are no use of strings to declare targets or to reference target dependencies. It is just plain and simple C# in a type safe manner.

Question is, can we do even simpler? Written as a C# script without the need for a host console application or a special runner. 

The syntax I was aiming for was something like

```c#
Step Compile = () = WriteLine(nameof(Compile));

Step Pack = () => 
{
    Compile();
    WriteLine(nameof(Pack));
}

Step Test = () => 
{
    Compile();
    WriteLine(nameof(Test));
}

Step Full = () =>
{
    Pack();
    Test();
}

await ExecuteSteps(Args);
```

As we can see there is no special DSL in this syntax. A "Step" is just a delegate where the body of the method becomes extremely simple. Dependencies to other steps are specified by just calling the steps we depend upon.

The step itself is represented by a delegate `public delegate void Step()` and `dotnet-steps` reflects over these delegates and figures out what step(s) to execute based on the arguments passed to the script. 

### Reflecting "this"

As we can see in the `dotnet-steps` example there is no class that contains the steps. They are written as top level fields in the script. That is a very powerful aspect of C# scripting in general. We can just start to write code without any class containing it. 

The first challenge here is obtaining a list of available steps since we don't really have a class type for which we can use as a starting point for reflection. We can't use `this` either since C# scripting does not allow the `this` keyword in the top level portion of a script.

So where does this top level step fields end up? They don't belong to a class ...or do they?

It is a matter of fact that they actually do. It is just hidden for us. Top level members gets compiled as members of a class name `sumbisson#0`  which from the compilers point of view is just normal C# code. But how do we get access to this type and preferably also the instance of "this" represented as an instance of `submission#0`?

The trick here is to use a top level lambda to provide this information to us.

```c#
Action stepsDummyaction = () => StepsDummy();
void StepsDummy() { }
```

This `stepsDummyAction` is also a top level field so that it belongs to `submission#0` as well. It is not static so it must have a `target`, right?  Let's try this.

```c#
Action stepsDummyaction = () => StepsDummy();
void StepsDummy() {}
WriteLine(stepsDummyaction.Target.GetType());
```

And the result

```
Submission#0
```

Success! We now have access to `Submission#0` instance through the `Target` property of our dummy action. And we have access to the `Submission#0` type which again makes it possible to reflect over  `Step` fields declared in the `Submission#0` type.

So now we can write code like 

```c#
private static FieldInfo[] GetStepFields<TStep>()
{
	return _submissionType.GetFields().Where(f => f.FieldType == typeof(TStep)).ToArray();
}
```

In order to get the actual `Step` delegate we need to get the value from each `Step` field. Unless the field is declared as static, we also need the `Submission#0` instance to get the field value.

```c#
private static TStep GetStepDelegate<TStep>(FieldInfo stepField)
{
	return (TStep)(stepField.IsStatic ? stepField.GetValue(null) : stepField.GetValue(_submission));
}
```

> Note: These methods are generic so that they can work for different delegate types. We also have an `AsyncStep` for dealing with async steps.

### Executing steps

Now that we have access to the `Step` delegates, the next part is executing them which is pretty trivial by itself, but we also need to track the duration of each step so that we can present a nice summary report at the end of execution. 

Consider the following two steps.

```c#
Step step1 = () => WriteLine(nameof(step1));

Step step2 = () => 
{
    step1();
    WriteLine(nameof(step1));
};
```

If we execute `step2` we will first execute `step1` and then the rest of the code in `step2`. And remember that we still want to have `step1` appear in the summary report as a separate entry with its own duration measurement.

```c#
---------------------------------------------------------------------
Steps Summary
---------------------------------------------------------------------
Step                Duration           Total
-----               ----------------   ----------------
step1               00:00:00.0009179   00:00:00.0009179
step2               00:00:00.0008100   00:00:00.0017279
---------------------------------------------------------------------
Total               00:00:00.0017279
```

How can we possibly intercept the call to `step1` from within `step2` ? 

### Decorating Steps

Remember that these `Step` delegates are just instance fields on the underlying `Submission#0` instance? So what if we set these `Step` field values to another `Step`. A `Step` that wraps the original `Step` around a `Stopwatch`?

```c#
Step wrappedSted = () = 
{
	var stopWatch = Stopwatch.StartNew();
    step(); // This is the original step declared in the script
    stopWatch.Stop();
}
```

All we really need to do now is to set the field value to this wrapped `Step` instead.

```c#
var stepFields = GetStepFields<Step>();
foreach (var stepField in stepFields)
{
    var step = GetStepDelegate<Step>(stepField);
    Step wrappedStep = () =>
    {
        StepResult stepresult = PushStepResultOntoCallStack(stepField);
        var stopWatch = Stopwatch.StartNew();
        step();
        stopWatch.Stop();
        PopCallStackAndUpdateDurations(stepresult, stopWatch);
    };
    stepField.SetValue(stepField.IsStatic ? null : _submission, wrappedStep);
}
```

Okay, what is this `PushStepResultOntoCallStack` and `PopCallStackAndUpdateDurations` methods?

First of all, the `StepResult` class is just a simple class that contains information about the executed step.

```c#
public class StepResult
{
    public StepResult(string name, TimeSpan duration, TimeSpan totalDuration)
    {
        Name = name;
        Duration = duration;
        TotalDuration = totalDuration;
    }

    public string Name { get; }
    public TimeSpan Duration { get; set; }
    public TimeSpan TotalDuration { get; set; }
}
```

So for each executed step, we record the `Duration` which is the time spent in a step excluding the time spent calling other steps. The `TotalDuration` is the time spent executing the step including the time spent calling other steps. 

For this to work we need some kind of call stack so that we can keep track of the exclusive time being spent in each step. 

```c#
private static StepResult PushStepResultOntoCallStack(FieldInfo stepField)
{
	var stepresult = new StepResult(stepField.Name, TimeSpan.Zero, TimeSpan.Zero);
    _callStack.Push(stepresult);
    return stepresult;
}
```

When the step has executed, we pop the current `StepResult` off the stack and update the duration of the calling step.

```c#
private static void PopCallStackAndUpdateDurations(StepResult stepresult, Stopwatch stopWatch)
{
    var durationForThisStep = stopWatch.Elapsed;
    stepresult.TotalDuration = durationForThisStep;
    _results.Add(_callStack.Pop());

    if (_callStack.Count > 0)
    {
        var callingStep = _callStack.Peek();
        callingStep.Duration = callingStep.Duration.Subtract(durationForThisStep);
    }
    stepresult.Duration = stepresult.Duration.Add(durationForThisStep);
}
```







 

* Reflection
* CallStack
* Decorating fields
* Testing 
* Script packag

