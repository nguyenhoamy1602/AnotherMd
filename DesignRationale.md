Group 109

Vu Le, Tuong Pham, Nguyen Hoa My
# Design Rationale - Learning to Escape
## Introduction
In this assignment, we are tasked with creating a car auto-controller that is able to successfully traverse the map and its traps. It must also be capable of safely: 
1. exploring the map and locating the keys 
2. retrieving the keys in order 
3. making its way to the exit
## The Situation
The original code-base provide us with ```AIController``` that extends from the abstract ```Car  Controller``` class which has basic follow-wall behaviour to get out of the maze.
After investigating, we concludes that the ```AIController``` class has too many functions, to help make the program easier to maintain and increase cohesion, multiple classes need to be added with their unique functions. We design our package keeping in mind General responsibility assignment software patterns (GRASP) and Gang of Four Design Patterns.
## Our Escape Strategy
The car follow a left Wall-Following Strategy to traverse to the exit and find the key. In addition, when the car stay in the health trap, it always stay there until its health is maximum. Anytime Health metrics fall below a predefined threshold and there is a path to Health Trap in the car’s vision, it will directly go to the Health Trap to stay there until it fully recovers. Also, if the car is following the wall and crossing over a health trap, it will come back to that health trap to recover before continue moving.


## MySensor
We decided to separate all functions dealing with getting informations from environment into an independent class named ```MySensor```. This class achieves Information Expert pattern since it is the only class directly observe the environment around the cars, i.e. walls, traps and finish zone. With all its functions dealing with tiles surrounding the car, many of them use the same functions that also contains within MySensor. The high cohesion it achieved allowed other classes to utilized it easily without fear of instability. Low coupling is also achieved as the module prohibit others from accessing directly its attributes through the use of interfaces, which will be discussed further in Observer pattern section.

## Direction Utils
This class is purely designed as a pure fabrication and defines all the functions for controlling and facilitating the usage of direction in this simulation. By defining this class, we can increase the cohesion of MySensor class which was originally responsible for the direction management. Inevitably, we have slightly increased the coupling of the system since almost every class uses the functions in DirectionUtils. However, all functions in this DirectionUtils class are fixed and static, and so it totally does not affect any properties in other classes so that this trivial increase in coupling can be safely neglected.


## Balance Controller
To adjust the traversing direction of the car, a balancecontroller package is also created. This package helps increase the cohesion of the  entire module as it has one main function of ensuring, the car is moving straight in its direction. Similar to pathfollower, a Strategy pattern with Polymorphism is applied, with BalanceStrategy implementing BalanceBehaviour, then both LeftBalance and RightBalance extends from BalanceStrategy. This is because LeftBalance and RightBalance have basically the same functions with different logic.

## MyPathFinder
### MyPathFinder Introduction
As the name suggests, this package contains all necessary classes and interfaces used to find a path which can be deemed as a navigation guide to a particular coordinate.  This package contains an abstract class ```BasicPathFinding``` which defines a couple of common and basic functionalities for the two main children classes ```BestPathFinding``` and ```SinglePathFinding```. Also, there is an interface ```IPathFindingStrategy```, which defines an abstract function getPath for all classes in this package.

### Design Pattern Choice and Implementation Clarification
The implementation and algorithms used for the PathFinding are first considered in order to have a clear insight into the design pattern determination. In particular, the Dijkstra searching algorithm is implemented in all path finding functions. As the sensor can detect many coordinates for a particular type of tile at the same time, and the car just needs to go to one coordinate, we need to find the “best” tile amongst the set of found locations. In this case, we consider “best” tile as the tile that has the minimum distance and minimum traps to the original location. Therefore, Dijkstra would be the suitable algorithm since it can calculate the distance from the original coordinate to the rest coordinates within the car’s vision.
After finish choosing the algorithm for the path finder, we recognized an interesting pattern that can be applied to this design. In particular, in the initial design, the class BestPathFinding had to implement the Dijkstra algorithm and backtrack function to find the path to each coordinate that has the minimum distance. After that, it has to check whether each path is reasonable and the car is able to go through it. It is clear that the responsibility property for the BestPathFinding is relatively high. Therefore, a new class SinglePathFinding is created to help the BestPathFinding with backtrack function. The interesting thing is SinglePathFinding can be considered as variables of the BestPathFinding, but from the general sense, its behavior is getting the path, which is the same as BestPathFinding. Therefore, the most suitable design pattern for this scenario is the Composite Pattern. In Figure x, we can see that both BestPathFinding and SinglePathFinding class extends the BasicPathFinding class which implements the IPathFindingStrategy interface, and the BestPathFinding is responsible for creating the SinglePathFinding when it needs. 



![Figure 1. Composite Pattern in MyPathFinder package](https://github.com/iamnhvt/Part-C/blob/master/finder.png)


By applying this pattern, we do not need to create a new strategy for SinglePathFinding. Instead, we can make two classes be the same in terms of behaviors while still maintain the association between them and reduce the responsibility for the BestPathFinding.

## MyFollower
### Package Introduction
This package is purely designed for controlling the driving behavior of the car. Most of the escaping strategies stated at the beginning will be implemented in this package. In general, there are two main classes, namely WallFollower and PathFollower, and both classes implement the FollowBehavior strategy. The WallFollower defines and implements the behavior to make the car merely keep following the wall until it dies or it detects a tile to go to. On the other hand, the PathFollower is much more complicated as it has to follow a planned path. The path given to the PathFollower is completely generated by the BestPathFinding class so that these two classes are highly associated with each other. The class diagram of the design is presented in Figure 2.
### Design Pattern Choice and Declaration
Both GRASP and strategy pattern are applied in this package. The strategy pattern is easily observed since there are two distinct movement strategies for the car, and each of them implements the functions defined by IFollowBehavior interface. However, each class is associated and is a creator to each other. In specific, each instance of these two classes is a session, and when it finishes its job, it has to find another follower to keep driving the car. For example, when a car is following the war, and it finds a lava trap that contains the next key, it then terminates the current wall-following behavior, create a new instance of PathFollower, and put it in the follower queue. The same thing happens when PathFollower instance terminates. The current design increases the coupling of the system since every class in this package is connected to and uses each other. The association amongst all path followers is inevitable as it follows our predefined strategies. We can solve this problem by creating a new class that is responsible for arranging the path followers order and manage the queue. However, the current design is enough for the purpose of this simulation as there are just two kinds of path follower, and so the effect of high coupling to the system is still trivial.


![Figure 2. Design for package pathFollower](https://github.com/iamnhvt/Part-C/blob/master/newfollower.png)

The second design decision made in this package is Polymorphism in class WallFollower. During the development and testing process, we found that there is a need for two behaviors of WallFollower. In particular, one behavior prefers right turning and the other prefers left turning. Since the only difference between the two is the turning behavior, all the common functionality and variables are included in the parent WallFollower class, and they will override the update function to implement their own turning behavior. From the general sense, we can do better by applying turning strategy, and the wall follower just simply calls this strategy to switch its turning behavior. However, when we attempted to implement this design, the turning behavior uses both MyAIController and MySensor class to perform the turning movement so that it significantly increases the coupling of the design. Therefore, applying polymorphism to delegate the turning function to its children would be more appropriate in this case.


## Attempted Observer Pattern
We attempted to implement the communication between MySensor and WallFollower using observer patterns with two interfaces, iTrapListener and iWallListener. MySensor acts as a publisher and broadcasts event to its subscribers. Whenever it detects meaningful events, MySensor publishes those tiles locations to each of its listeners. There are two lists of subscribers for two separate events, trap detection and wall detection, and each can be registered dynamically with classes that are interested. Wall-detecting events are published constantly to update the status of wall obstacles to subscribers, while trap-detecting events are only broadcasted once at least one trap is detected. Each type of traps triggers separate events, and each contains a list of  coordinates corresponding to the locations of the traps. This way all subscribers can get the update instantly as well as differentiate between each type of traps. Overall, the pattern reduces coupling in the system as other classes do not have to call many functions of MySensor, lowering their interdependency.

## Conclusion
All in all, we have applied various design patterns to our mycontroller package including Strategy, Composite, Polymorphism, Pure Fabrication, and Observer to increase cohesion and reduce coupling. We have also employed Dijkstra algorithm to help our controller get out of the maze in the fastest way possible. Although it takes the car multiple drive around the maze to get out, the main focus of this task is not how fast but how reliable in achieving the objective. As a result, we are able to escape the maze without having to employ too complicated algorithm. 



