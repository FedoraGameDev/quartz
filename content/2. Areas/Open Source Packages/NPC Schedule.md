# NPC Scheduling System

#### Idea
Game-wide manager that holds all available tasks. NPCs will request tasks for a specific time slot and choose one that's available. The tasks will need filters, priorities, and requirements. NPCs should be able to fill in their schedules ahead of time. Some tasks can have multiple people attend and the npcs will do the task together. Tasks that allow multiple people can also have a minimum requirement. Anyone who applied for the task will have to reschedule if the minimum requirement is not met. Schedules need to be serializeable so schedules will remain set through loads.


![Class Diagram](3.%20Resources/Packages/NPC%20Schedule%20System/Class%20Diagram.md)


### Quick Start
In order to use this package, 5 requirements need to be met.
1. Implement `IContext`.
5. Create at least 1 implementation of `ITask`.

For the examples below, we'll assume I have some class structure explained abstractly as;
- class NPC
	- string Name { get; }
	- GameObject GameObject { get; }


#### IContext
This interface will be used for filtering the tasks. If you have a task that a specific NPC can do, you'll want to add a reference to the NPC class here. Whenever the meta data is required, you will need to fill in the references in a constructor.
```cs
public class Context : IContext
{
	public NPC Character { get; set; }
}
```

When we create an instance of `MetaData`, we can assign the Character:
```cs
NPC myNPC;
Context context = new Context(){ Character: myNPC }
```

#### ITask
This is where the meat of your code will be. `ITask` will be the interface implemented by all the tasks you can think of. For now, I'm just going to make a simple task for an NPC to stand at a specific location.
```cs
public class StandHereTask : ITask
{
	[SerializeField] Vector3 _position;
	
	public void DoTask(IContext context)
	{
		Context ctx = context as Context;
		ctx.Character.GameObject.Transform.Position = _position;
	}
}
```

### Additional Learning
#### ITaskFilter
Task filters are there to ensure only specific characters can do certain tasks. This is, of course, expandable to any criteria you may need as long as the data is stored in your `Context`.
Here we create an implementation specifically for the `MetaData` type you just created. You can have as many filter types as you want. Here, I'll create one for filtering against a specific NPC.
```cs
public class NPCTaskFilter : ITaskFilter<Context>
{
	[SerializeField] NPC _character;
	
	public bool IsValid<Context>(Context context)
	{
		return _character.Name == context.Character.Name;
	}
}
```
You could also easily just create an empty implementation that simply returns true in `IsValid`.
```cs
public class PassTaskFilter : ITaskFilter<Context>
{
	public bool IsValid<Context>(Context context) => true;
}
```

#### ITimeSolver and ITimeFrame
These will both refer to whatever system you're using to manage time. I have a package called `GameTime` that allows you to create custom date-time formats in the inspector which is why I needed some way to define these outside of this system.

With `ITimeSolver`, we just need to take the date/time value from the time system in place and convert it to a numerical value that can be compared against. In my example layout, minutes, hours, days, and months all fit within a single `byte` respectively, and the year fits in a `ushort`. If we take these values and stack them together, we'll take up 48 bits which fits within the 64 bits afforded to us with a `ulong`.
```cs
public class TimeSolver : ITimeSolver
{
	public ulong TimeValue {
		get {
			List<byte> bytes = new List<byte>(8);
			bytes.add(GameDateTime.Minute);
			bytes.add(GameDateTime.Hour);
			bytes.add(GameDateTime.Day);
			bytes.add(GameDateTime.Month);
			bytes.add(BitConverter.GetBytes(GameDateTime.Year));
			return BitConverter.ToUint64(bytes.ToArray(), 0);
		}
	}
}
```

#### ITaskPoolItem
By making `ItaskPoolItem` generic, we can force it to have access to the `MetaData` class, as well. This is helpful when dealing with the filters. With Odin Inspector, interface references can be serialized. This is my preferred method.
```cs
public class TaskPoolItem : ITaskPoolItem<Context>
{
		public ITask Task => _task;
		public ITimeFrame TimeFrame => _timeFrame;
		public ITaskFilter<Context>[] TaskFilters => _taskFilters;
		public int Priority => _priority;
		public int MinAttendies => _minAttendies;
		public int MaxAttendies => _maxAttendies;
		
		[OdinSerialize] ITask _task;
		[OdinSerialize] ITimeFrame _timeFrame;
		[OdinSerialize] ITaskFilter<Context>[] _taskFilters;
		[SerializeField] int _priority;
		[SerializeField] int _minAttendies;
		[SerializeField] int _maxAttendies;
}
```