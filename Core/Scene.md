# Scene

```csharp
public enum SceneType
{
  Process = 0,
  Manager = 1,
  Realm = 2,
  Gate = 3,
  Http = 4,
  Location = 5,
  Map = 6,

  // 客户端Model层
  Client = 30,
}
```




```csharp
namespace ETModel
{
	/// <summary>
	/// 之前每个功能是一个进程，比如realm gate location map，
	/// 现在改成每个功能是一个Scene，一个Scene可以放到一个进程中
	/// - 所有的Scene放在一个进程就变成了AllServer模式
	/// - 客户端gamescene是永久存在的,再搞个scene挂在gamescene下面作为当前进入的场景，切换场景的时候删除这个scene再创建一个scene。
	/// </summary>
	public sealed class Scene: Entity
	{
		/// <summary>
		/// Process = 0,Manager = 1,Realm = 2,Gate = 3,Http = 4,Location = 5,Map = 6,客户端Model层 Client = 30,
		/// 也可以是其他类型的, 例如一个房间,一个副本
		/// </summary>
		public SceneType SceneType { get; set; }

		public string Name { get; set; }

		public Scene Get(long id)
		{
			return (Scene)this.Children[id];
		}

		/// <summary>
		/// new关键字,隐藏基类的Domain的get/set方法,
		/// 使派生类不能访问基类的方法
		/// </summary>
		public new Entity Domain
		{
			get
			{
				return this.domain;
			}
			set
			{
				// domain是protected可以被派生类修改
				this.domain = value;
			}
		}

		public new Entity Parent
		{
			get
			{
				return this.parent;
			}
			set
			{
				this.parent = value;
				this.parent.Children.Add(this.Id, this);
			}
		}
	}
}
```
