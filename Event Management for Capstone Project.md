# Event Management for Capstone Project

Our capstone project is a Cooperative Action-RPG (spell casting) game made with Unity, and I worked on networking (Photon Network), event system, AI behavior tree, control, and camera etc.

When we initially started our project, there wasn't any event management, and things like UI health updates, we were doing them every frame, which was an efficiency killer. More importantly, it was hard to communicate between classes without an event system, as you would have to put references everywhere, and everything would be deeply coupled.

The idea of event system is fairly simple, whenever something happens, we create an event with certain information and send it over to the ones who care about this event (observer pattern). What I want is some kind of manager that handles all the processing for both local events and networking events, and provide a uniformed API for other programmers in our team to use with ease. All they have to know is some public APIs to queue events, listen to events, and handle them.

However, there were several problems when I was implementing it.
********************************
### **Problem 1:** Passing different information around.
**Solution:** Polymorphism. By creating an empty class EventArgs, and inheriting the actual classes from it, we can pass all the inherited classes as EventArgs.
``` C#
  [Serializable]
  public class EA_Local_StatChange : EventArgs {
      ...
  }
```
********************************
### **Problem 2:** Dynamically creates new listener lists for new Event Types.
**Solution:** Dictionary. Every time a new event type is being listened, we check if the event type is already existing, if not, create a new list for it. Using the Action wrapper so that we don't have declare the delegates manually.
``` C#
  // lists of handlers of event types
  Dictionary<EventType, List<Action<EventArgs>>> listeners_local;
  Queue eventQueue_local;
  
  public void AddListener_Local(EventType type, Action<EventArgs> handler) {
      if (!listeners_local.ContainsKey(type)) {
          listeners_local[type] = new List<Action<EventArgs>>();
      }
      listeners_local[type].Add(handler);
  }

  public bool QueueEvent_Local(EventType type, EventArgs args) {
      if (!listeners_local.ContainsKey(type)) {
          //Debug.Log("No listeners for " + type.ToString());
          return false;
      }
      eventQueue_local.Enqueue(new GameEvent_Local(type, args));
      return true;
  }
```
********************************
## **Problem 3:** Network events, the evil one. 
We are using the PhotonNetwork, and besides RPC, there is a convenient function to send an event over the network: 
``` C#
staic bool RaiseEvent(byte eventCode, object eventContent, bool sendReliable, RaiseEventOptions options)
``` 
So whenever we ask the EventManager to queue a network event, it's gonna call this function, and it will also catch this event by registering itself to **PhotonNetwork.OnEventCall**, and then dispatches it to all listeners. I thought it was brilliant, until later I realized that PhotonNetwork could not serialize/deserialize custom types, not even Vector3, and I did a lot research and disappointly found that there is no easy way other than serializing and deserializing them myself. So everytime my group wants a new type of EventArgs, I would have to write these two functions manually:
``` C#
private short SerializeEA_Network_EnemyDeath(StreamBuffer outStream, object customobject) {
    EA_Network_EnemyDeath eaObj = (EA_Network_EnemyDeath)customobject;
    int index = 0;
    byte[] bytes = ObjectToByteArray(eaObj);
    Protocol.Serialize(eaObj.currencyDrop, bytes, ref index);
    // Well, it doesn't support Vector3 either, we have to do it the x,y,z way
    Protocol.Serialize(eaObj.dropPositionX, bytes, ref index);
    ...

    outStream.Write(bytes, 0, bytes.Length);
    return (short)bytes.Length;
}

private object DeserializeEA_Network_EnemyDeath(StreamBuffer inStream, short length) {
    EA_Network_EnemyDeath eaObj = new EA_Network_EnemyDeath();
    byte[] bytes = ObjectToByteArray(eaObj);
    inStream.Read(bytes, 0, bytes.Length);
    int index = 0;
    Protocol.Deserialize(out eaObj.currencyDrop, bytes, ref index);
    Protocol.Deserialize(out eaObj.dropPositionX, bytes, ref index);
    ...

    return eaObj;
}
```
Then register them by
``` C#
PhotonPeer.RegisterType(typeof(EA_Network_EnemyDeath), 1, SerializeEA_Network_EnemyDeath, DeserializeEA_Network_EnemyDeath);
```
As you could imagine, it's a pain.

**Solution:** We lived with that for a while since it worked and we have a tight schedule. Recently, it suddenly occurred to me that, maybe, instead of serializing them, I can convert the whole object into a JSON string, pass it along, and convert it back, as **string** is a primitive type that is serializable on PhotonNetwork. And there is a handy class in Unity or C# that does the conversion for me.
``` C#
public bool QueueEvent_Network(EventType type, EventArgs content, 
                               bool reliable = true, RaiseEventOptions options = null) {
    if (!listeners_network.ContainsKey(type)) {
        Debug.LogWarning("No listeners for " + type.ToString());
        return false;
    }

    if (options == null) {
        options = new RaiseEventOptions();
        // by default, no caching, no interest group, no target actors, no sequence channel, send to all
        options.CachingOption = EventCaching.DoNotCache;
        options.InterestGroup = 0;
        options.TargetActors = null;
        options.Receivers = ReceiverGroup.All;
        options.SequenceChannel = 0;
    }
    // Here is the key
    var contentJson = JsonUtility.ToJson(content);
    // GameEvent_Network is just an internal container for convenience
    eventQueue_network.Enqueue(new GameEvent_Network(type, contentJson, reliable, options));
    return true;
}

public void OnEnable() {
    PhotonNetwork.OnEventCall += PhotonEventDispatcher;
}

// EventManager is receiving all photon events, and dispatch them to the actual listeners
private void PhotonEventDispatcher(byte eventCode, object content, int senderID) {
    EventType type = (EventType)eventCode;

    foreach (var handler in listeners_network[type]) {
        handler(content.ToString());
    }
}
```
And in the actual listener,
``` C#
public void OnEnemyDeath(string argsJson) {
    var args = JsonUtility.FromJson<EA_Network_EnemyDeath>(argsJson);
    if (args == null)
        return;
    ...
}
```
********************************
All my team need to know are basically three sets of functions:
``` C#
AddListener_Local(EventType type, Action<EventArgs> handler)
AddListener_Network(EventType type, Action<string> handler)

QueueEvent_Local(EventType type, EventArgs args)
QueueEvent_Network(EventType type, EventArgs content, bool reliable = true, RaiseEventOptions options = null)

// and actually implement the handlers
// local
void OnChangeSkill_Local(EventArgs args)
// network
void OnEnemyDeath(string argsJson)
```
Of course it can be improved in every way, like it doesn't really need two lists for both local and network events, but that's a trivial problem. Nevertheless, I learnt a lot from implementing this.
