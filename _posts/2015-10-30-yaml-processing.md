---
layout: post
title: YAML parsing in C++ using yaml-cpp
---


In this post I want to discuss how I will use yaml-cpp for parsing a YAML document.
The document I intend to parse contains description about what I call "command 
primitives" for PANDUbot. Command primitives are atomic commands that the robot
controller's executive layer can directly execute. For example a complex command
like "Goto vending machine and Get snickers" has two command primitives: "Goto"
and "Get" that are connected by the word "and". The thing following the command
primitive defines the goal of the robot for that particular command primitive. 
The YAML file I am parsing will actually contain a list of the command primitives
PANDUbot can recognize and the recognized goals.

```YAML
goto:
  - vending_machine: [0,3,4]
  - dustbin: [1,12,0]
  - bump_space: [22,3,4]
    
get: # Represent scalars as lists so that the parsing can be done as "vector"
  - snickers: [0]
  - kit_kat: [1]
  - root_beer: [3]
  ```

The "goto" and "get" primitives are represented as separate nodes in the YAML file.
Each of them has a list of associative arrays that convert the user commanded goal
to a numeric value that the robot can work with - so for goto/vending_machine key,
the value is a vector of coordinates of the vending machine in the map. 
The key:value pairs for each primitive can stored as a std::vector of std::maps 
in c++. I chose std::map over std::pair because I would like to index the value 
of the associative array by its corresponding key. 

Notes on YAML Parsing:
  YAML::Node root = YAML::Load ("test.yaml") // this loads a scalar node with the name  
                                        // of the file being read - test.yaml.:w
Everything in the yaml document is interpreted as a node by the yaml-cpp library.
To actually load a file for parsing use:
  YAML::Node root = YAML::LoadFile("test.yaml")
This loads an iterable object having size equal to the number of base nodes. For eg:
N1:
  - x:1
  - y:2
N2: 
  - x:4
  - y:5
This would have two base nodes which you can iterate over.
To view the number of base nodes just do:
  cout<<root.size();
Note that the nodes in the above yaml file are such that the base nodes are associative 
arrays such that each node maps a key (N1/N2) to a list of associative arrays. the
document itself is a list of the base associative arrays

So basically to parse the values of N1 you do the following:
  YAML::Node root = YAML::Load("test.yaml"); //associative array of two elements
  YAML::Node first_basenode = root["N1"]; // list of associative arrays
  std::vector<map<string,int> > first_basenode_entry = first_basenode.as< std::vector <map<string,int>> > ();

Note that the associative arrays and lists are automatically converted to std::maps
and std::vectors respectively. So you can see that anything in the document: list,
associative array or even a scalar can be represented as a YAML::Node which can 
then be cast into an appropriate type. You can also define operators to make this
casting process more customised to your data structures by overloading operators.

The library also provides iterators to iterate over the nodes which can be used 
as follows:
  YAML::const_iterator itr = root.begin();
  for(;itr!=root.end();itr++){
    YAML::Node entry =  *itr; // this  would be N1, N2 ...
    /*process here*/
  }

Here's the code that I finally ended up with:

```c++
// Atomic operator to save a single entry in the yaml file as a hashmap 
template<class T>
void operator << (map<string,T>& output, YAML::Node& node) {
  YAML::const_iterator itr = node.begin();
  string key = itr->first.as<string>();
  T value = itr->second.as<T>();
  output[key] = value;
};

class CommandParser {
  private:
    string filename_;
    YAML::Node root_;
  
  public: 
    CommandParser(string filename);
    map <string, vector<float> > ConvertTaskToExecutiveGoal(pair<string,string> task); 
};

```
```c++
#include "read_yaml.hpp"

CommandParser::CommandParser(string filename)
: filename_(filename) 
{ 
  root_ = YAML::LoadFile(filename_);
}


map<string, vector<float> > CommandParser::ConvertTaskToExecutiveGoal(pair<string,string> task) {
  string predicate = task.first;
  string subject = task.second;
  map<string, vector<float> > return_goal;
  bool predicate_found = false;
  bool subject_found = false;
  for(YAML::const_iterator root_itr = root_.begin(); root_itr!=root_.end(); root_itr++){
    string command_primitive = root_itr->first.as<string>();
    if(command_primitive==predicate){
      predicate_found = true;
      YAML::Node possible_goals = root_[command_primitive];
      for(YAML::const_iterator goal_itr=possible_goals.begin(); goal_itr!=possible_goals.end(); goal_itr++){
        YAML::Node goal = *goal_itr; // matched associative array in YAML file
        YAML::const_iterator goal_map = goal.begin(); // use this to get key for the array 
        string goal_name = goal_map->first.as<string>();
        if(goal_name==subject){
          subject_found = true;
          return_goal << goal;//overloaded operator see header file
          return return_goal;
        }
      }
    } else {continue;}
  
  }
  
}
```


