# 🏭 FIEAGameEngine_Documentation
This is the documentation of the custom game engine that I built during my time at Florida Interactive Entertainment Academy. It was a semester long project to make a game engine from the ground up. The actual repository has been made private to prevent copying of code and I can make it available on request.

---
## About the project
This game engine is built from the bottom up. It is a designed on the founding principles of a modern game engine, that ingests content written in JSON creates C++ objects internally with a 2 way binding.
It uses a lot of modern C++ techniques and design patterns for the implementation of its various components.

## Documentation
📘 [View Full Documentation (Doxygen)](https://shauryapandey.github.io/FIEAGameEngine_Documentation/)

## 🧠 Key Components and highlights of the game engine
You may review the full documentation or go through the condensed description of the key concepts that are central to the game engine and make it content driven.

- `Datum` : A runtime-bound container for zero or more elements of the same type analogous to the value for a member field of an object, including arrays. Each Datum object represents an array, and all of the elements in that array have the same type. Different Datum objects, however, can store values of different types.
The datum holds a union of these various types.
  ```C
          union DataUnion
        {
            int32_t* _Int;
            float* _Float;
            string* _String;
            vector* _Vec4;
            matrix* _Mat4x4;
            /*has them but does not own*/
            RTTI** _Rtti;
            Scope** _Scope;
        };
  ```
- `Scope` : The second piece of the Datum & Scope data subsystem which forms a dynamic hierarchical database. Scope objects are tables that create dictionary of name-value pairs where Datum objects are the values. Each entry in a Scope table has a name and a Datum, where the Datum represents an array of values of a single type. Furthermore, an entry in a Scope table can refer to another Scope table and thereby provides the means to create user-defined types which are a Datum type. So the Datum & Scope classes form a recursive pair: Scopes are tables of Datum, some of which can be other tables (i.e. Scopes).  Also, we will ensure that each Scope has a pointer to its parent, thus forming a navigable tree of Scopes. 

Some important API endpoints from Scope that drive home how Scope is intended to be used.
```C
  Datum* Find(const string& key); // return the Datum at the requested key, or nullptr if not found
  Datum* Search(const string& key, Scope** outputScope = nullptr);
  Datum& Append(const string& key); // return a reference to the Datum at this key. create a new Datum, if one is not found
  void RemoveDatum(const string& key);
  Scope& AppendScope(const string& key, Scope* scope = nullptr); // Add a reference to the provided Scope to a Table-type Datum at 'key'
  void Adopt(const string& key, Scope& child); // Assume parentage of the provided child
  Scope* GetParent() const;
  void SetParent(Scope& parent); // can assume that the Scope is currently a Root Scope
  [[nodiscard]] Scope* Orphan(); // remove parentage and return the pointer to the caller, since they will now own this Root Scope

  // search for a nested Scope which is the SAME (not just equal) as the provided scope and return the Datum holding it. Populate idx with the index within the Datum
  Datum* FindContainedScope(const Scope& child, size_t& idx);
  const Datum* FindContainedScope(const Scope& child, size_t& idx) const;

  bool IsAncestorOf(const Scope& descendent) const;
  bool IsDescendentOf(const Scope& ancestor) const;
```
- `Attributed`: With Scope and Datum, we have the ability to produce complex objects in a data-driven fashion. My goal, however, is more specific than just creating YASL (yet another scripting language); I want to bind the data in that language to native data within the game engine. To achieve this, the scope object must be glued directly to classes and objects in the engine’s native language (C++). Those native classes are defined at compile-time. Although the Content system allows us to create dynamic data structures at run-time, we need to express the “schema” at compile-time, to provide the required type information to mirror native classes.
Consider an example
```C
struct Foo final
{
     int32_t Health{ 0 };
     float DamagePerSecond{ 0.0f };
};
```
We want to expose this class (and objects of this class) to our scripting language. That means any time an object of type Foo is created, we want to create Scope objects which mirror those Foo objects. We want to make this “mirroring” easy to code. 
Attributed class enables custom C++ classes to have 2 way binding with their parent Scope. To create that mapping, the following are the key ideas.
```C
    struct Signature
    {
        const char* _Name;
        size_t _Offset;
        Datum::Type _Type;
        size_t _Count;
    };

    struct ClassDefintion
    {
        std::vector<Signature> _Signatures;
    };

    using GetClassDefintionFunc = ClassDefintion(*)(void);
```
Each class provides its vector of signatures to Attributed during construction. Attributed then uses these signatures to create the key value pairs in the Scope's table and externally binds the values to the derived class's member variable. The following snippet shows how the binding is done.
```C
    void Attributed::BindSignature(const Signature& signature, Datum& datum)
    {
        if (signature._Offset == 0)
        {
            //Storage type not set so can be internal if desired
            datum.SetType(signature._Type);
            if (signature._Count != 0)
            {
                datum.SetMaxSize(signature._Count);
            }
            return;
        }
        if (signature._Type == Datum::Type::Integer)
        {
            int32_t* ptr = (int32_t*)(((size_t)this) + signature._Offset);
            datum.SetStorage(ptr, signature._Count);
        }
        else if (signature._Type == Datum::Type::Float)
        {
            float* ptr = (float*)(((size_t)this) + signature._Offset);
            datum.SetStorage(ptr, signature._Count);
        }
        else if (signature._Type == Datum::Type::String)
        {
            string* ptr = (string*)(((size_t)this) + signature._Offset);
            datum.SetStorage(ptr, signature._Count);
        }
        else if (signature._Type == Datum::Type::Pointer)
        {
            RTTI** ptr = (RTTI**)(((size_t)this) + signature._Offset);
            datum.SetStorage(ptr, signature._Count);
        }
        else if (signature._Type == Datum::Type::Table)
        {
            Scope** ptr = (Scope**)(((size_t)this) + signature._Offset);
            datum.SetStorage(ptr, signature._Count);
        }
        else if (signature._Type == Datum::Type::Vector)
        {
            vector* ptr = (vector*)(((size_t)this) + signature._Offset);
            datum.SetStorage(ptr, signature._Count);
        }
        else if (signature._Type == Datum::Type::Matrix)
        {
            matrix* ptr = (matrix*)(((size_t)this) + signature._Offset);
            datum.SetStorage(ptr, signature._Count);
        }
    }
```
- Serialization: The other key step in the game engine is deserialization of user provided content to create the scope-datum objects. I utilize JsonCpp to provide basic JSON deserialization capabilities. There is an abstract `ParseWriter` that I use to write the deserialized objects. The interface `IParseHandler` decouples the Parser from the assorted code that needs to process the individual json values.
The Deserialize algorithm is as follows:
  - Iterate recursively over json key->value pairs
      - For each pair, invoke handlers in order, until one claims success in handling the key.
          - Likely, during the EnterKey or ExitKey, a handler returning true will interact with a ParseWriter to record the value in a way that makes sense
          - Handlers must know how to write to 1 or more ParseWriter types
          - If EnterKey returns true for a key->value pair, then we should ultimately call ExitKey when we are done processing that pair
              - Also, the key->value pair should not be given to other handlers... it has been "handled"... this is why the order of the handlers may be quite important.
          - When encountering a value that is an object, increase "depth" and iterate over the pairs within
          - When encountering a value that is an array, increase "depth," and iterate over the values within
        
Take a look at what a handler looks like. This is a float handler for handling float values.
```C
class FloatParseHandler final :public IParseHandler
{
    FIEA_HANDLER
public:
    FloatParseHandler() = default;
    ~FloatParseHandler() = default;

    virtual bool EnterKey(ParseWriter& writer, const std::string& key, const Json::Value& value) override;
    virtual void ExitKey(ParseWriter& writer, const std::string& key, const Json::Value& value) override;
private:

};

bool FloatParseHandler::EnterKey(ParseWriter& writer, const std::string& key, const Json::Value& value)
{
    if (value.type() == Json::ValueType::realValue)
    {
        ScopeWriter* scopeWriter = writer.As<ScopeWriter>();
        if (scopeWriter != nullptr)
        {
            //is key external storage in datum? then set external status
            if (scopeWriter->IsKeyStorageExternal(key))
            {
                scopeWriter->SetExternalDataTeller(HandlerId());
            }
            return scopeWriter->Append(key, value.asFloat());
        }
    }
    return false;
}

void FloatParseHandler::ExitKey(ParseWriter& writer, const std::string& key, const Json::Value& value)
{
    ScopeWriter* scopeWriter = writer.As<ScopeWriter>();
    if (scopeWriter != nullptr)
    {
        //is key external storage in datum? then reset it since this handler is exiting
        if (scopeWriter->IsKeyStorageExternal(key))
        {
            scopeWriter->ResetExternalDataTeller(HandlerId());
        }
    }
}
```
Like this there are plenty of handlers: `ActionHandler` : Action is an Attributed and are data defined behaviours. `ActionListHandler` : ActionList is a list of actions. `ArrayHandler`: Handles arrays, `GameObjectHandler` : Handles GameObject data, `IntParseHandler`: Handles integer data, `StringParseHandler`: Handles string data, `VectorHandler`: Handles vector data, `MatrixHandler` : Handles Matrix data, `ScopeParseHandler` : Handles scopes 

- `GameObject` : Extends Attributed and provides some "typical" object functionality
     - Positioning - uses a Transform struct so that GameObject's have a position in the world
     - Updating - GameObejct supports an Update method, which updates an Object and other GameObjects nested within it. We call them as Children GameObjects. GameObject can also have Actions attached to them and it will also call Update on its attached actions.
- `Action`: Is an Attributed the provides the users of the engine the ability to script behavior from content. It contains an Update function, returning a boolean for whether the Action is completed or ongoing. Update calls 3 virtual functions -
    - Init : optional initialization for the Action, return a boolean for whether to complete w/o Run.
    - Run :  perform the Action, return a boolean for whether completed
    - Cleanup : optional cleanup from any work that was done in the init.
GameObjects may have Actions attached to it, which execute on each Update. Actions remain attached to the object until they return "completed" from their Update. Action can be extended to make custom actions as per the requirements of the game. Some Actions provided out of the box by the engine are :
   - `ActionList`: Action List class owns a bunch of actions and its purpose is to execute those actions. The action list finishes when the owned actions finish.
   - `DelayedAction`: Waits until the requisite time has passed and then execute the contained actions until their completion. It provides a repeat count, which can default to 1.
   - `IncrementAction`:  This action lets user specify which datum/attribute in parent scope to increment.
   - `TimedAction`: Execute each item in the Action List until a certain amount of time has passed.
   - `WhileActionList`: This action helps the user define a loop in content.
   - `PreambleAction`: This action intializes the value of a variable. Value is specified in the action by the user.
   - `IntegerConditionalAction`: User specifies the key in a parent scope to look for, user also specifies a value. The action compares the user specified value against the value that is mapped to the key. This action is used for conditional exiting from a loop. User also specifies an IsMax attribute to tell the action whether it should do a < check or > check.
```C
class DelayedAction : public ActionList 
{
     RTTI_DECLARATIONS(DelayedAction, ActionList);
 public:
     static Attributed::ClassDefintion GetClassDefinition();
     DelayedAction();
     DelayedAction(int32_t delay, int32_t repeatCount);
     //Rule of 5
     DelayedAction(const DelayedAction& rhs) = default;
     DelayedAction(DelayedAction&& rhs) noexcept = default;
     DelayedAction& operator=(const DelayedAction& rhs) = default;
     DelayedAction& operator=(DelayedAction&& rhs)  = default;

     virtual ~DelayedAction() ;

     //Rule of 7 equality check and Equals
 
     //Inherited functions from Attributed
     virtual DelayedAction* Clone() const override;

     //RTTI override
     virtual std::string ToString() const override;
     //virtual bool Equals(const RTTI* rhs) const override;
     virtual bool Init() override;
     virtual bool Run() override;
     virtual void Cleanup() override;
     virtual void OnReset() override;

     int32_t _Delay;
     int32_t _RepeatCount;

 private:
     int32_t _StartTime;
     int32_t _CurrentIteration;
 };
```

Here is an example of how data and behaviour is specified by the user in json. This json is deserialized by the parser with the help of the handlers and writer(s) creating the nested heirarcy of Scopes.
```json
std::istringstream simpleStringStream(R"json({
  "<Type>": "GameObject",
  "Health": 100,
  "Mana" : 100,
  "MaxHealth" : 200,
"Actions" :
{
"<Type>" : "ActionList|WhileActionList",
"Condition" : 
{
"<Type>" :"Action|IntegerConditionalAction",
"MaxValue" : 0,
"IsMax" : 0,
"ParentKey" : "Health"
},
  "Actions" :
[
{
        "<Type>" : "ActionList|DelayedAction",
        "Name" : "RecursiveDamageAction",
        "Delay" : 5,
        "RepeatCount" : 0,
        "Actions" : 
                {
                "<Type>" : "Action|IncrementAction",
                "DatumToModifyKey" : "Health",
                "Diff" : -50.0,
                "Name" : "DamageAction"
                }
        },
{
"<Type>" : "ActionList|DelayedAction",
"Name" : "RecursiveHealAction",
"Delay" : 8,
"RepeatCount" : 0,
"Actions" :
{
"<Type>" : "ActionList|WhileActionList",
"Condition" : 
{
"<Type>" : "Action|IntegerConditionalAction",
"ParentKey" : "Mana",
"MaxValue" : 0,
"IsMax" : 0
},
"Actions" :
[
{
"<Type>" : "Action|IncrementAction",
"Name" : "ManaReduceAction",
"DatumToModifyKey" : "Mana",
"Diff" : -10.0
},
{

"<Type>" : "ActionList|DelayedAction",
"Delay" : 1,
"RepeatCount" : 0,
"Name" : "SuperHealAction",
"Actions" : 
{
"<Type>" : "Action|IncrementAction",
"Name" : "HealthIncrementAction",
"DatumToModifyKey" : "Health",
"Diff" : 5.0
}
}
]
}
}
]
}
})json"s);

```

- Services: An engine will have many services. The engine has a `ServiceManager` that provides a consistent way to manage those services, while gatekeeping access to them through interfaces. The Services using the Serivice Locator design pattern automatically register with the Service Manager without the Manager needing to have any direct dependency on the service code. I have a few important services in the engine : `FactoryService`, `ContentService`,`ClockService`,`MemoryService`.

