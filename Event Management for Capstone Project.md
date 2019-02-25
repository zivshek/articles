# Event Management for Capstone Project

Our capstone project is a Cooperative Action-RPG (spell casting) game made with Unity, and I worked on networking (Photon Network), event system, AI behavior tree, control, and camera etc.

When we initially started our project, there wasn't any event management, and things like UI health updates, we were doing them every frame, which was an efficiency killer. More importantly, it was hard to communicate between classes without an event system, as you would have to put references everywhere, and everything would be deeply coupled.

The idea of event system is fairly simple, whenever something happens, we create an event with certain information and send it over to the ones who care about this event (observer pattern). What I want is some kind of manager that handles all the processing for both local events and networking events, and provide a universal API for other programmers in our team so that they can use it with ease.

**************************
* Listen to an event: 

**************************
* **Problem:** Passing different information around.
* **Solution** Polymorphism. By creating an empty class EventArgs, and inheriting the actual classes 
**************************
