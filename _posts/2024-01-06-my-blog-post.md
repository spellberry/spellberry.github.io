# Ability System for a Third Person Character Controller

Developing an ability system has been something that I've wanted to do since I've started to get more and more comfortable with programming. In the past 8 weeks, I've developed an ability system for a game with a third-person character controller. Below you will see a short showcase video of what the program offers:
- A simple third-person character controller with a directional camera and a crosshair for aiming.
- Three types of abilites that the demo character can use - Projectile, AOE and Blink/Dash  
- The ability to tweak parameters to your liking for the different abilities
- You can also create a new ability and give it to one of the characters in the world (you can also load in whatever abilities are already saved to any of the players)

<video width="500" height="500" controls>
  <source src="/videos/generalShowcase.mp4" type="video/mp4">
</video>

## Integrating Jolt Physics into the project

The main inspiration for this project was the battle-royale [Spellbreak](https://www.youtube.com/watch?v=6Qx7Y1cSUoM). Since the game was in 3D and allowed aiming with a crosshair, I had to use a 3D physics engine to replicate something similar. I decided to use [Jolt Physics](https://github.com/jrouwe/JoltPhysics). After looking into how to integrate it into my project, I started setting it up. The first thing was to properly initialize it:

```cpp
// Configuration
//-----------------------------------------------------------------------------
static constexpr uint cNumBodies = 10240;
static constexpr uint cNumBodyMutexes = 0;  // Autodetect
static constexpr uint cMaxBodyPairs = 65536;
static constexpr uint cMaxContactConstraints = 20480;

Demo::Demo()
{
    RegisterDefaultAllocator();

    // Create factory
    Factory::sInstance = new Factory;

    // Register physics types with the factory
    RegisterTypes();

    m_TempAllocator = new TempAllocatorImpl(32 * 1024 * 1024);
    m_JobSystem = new JobSystemThreadPool(cMaxPhysicsJobs, cMaxPhysicsBarriers, mMaxConcurrentJobs - 1);

    // Physics

    // Store old gravity
    Vec3 old_gravity = m_PhysicsSystem != nullptr ? m_PhysicsSystem->GetGravity() : Vec3(0, -9.81f, 0);

    // Setting up Jolt's Physics
    m_PhysicsSystem = new PhysicsSystem();
    m_PhysicsSystem->Init(cNumBodies, cNumBodyMutexes, cMaxBodyPairs, cMaxContactConstraints, m_BroadPhaseLayerInterface,
                          m_ObjectVsBroadPhaseLayerFilter, m_ObjectVsObjectLayerFilter);
    m_PhysicsSystem->SetPhysicsSettings(m_PhysicsSettings);

    m_PhysicsSystem->SetGravity(old_gravity);
    m_PhysicsSystem->SetContactListener(&m_ContactListener);

    m_BodyInterface = &m_PhysicsSystem->GetBodyInterface();
    m_ContactListener.SetBodyInterface(m_BodyInterface);

    // Initializing the debug renderer
#ifdef DEBUG
    m_WireframeRenderer = new JPH::WireframeRenderer();
#endif // DEBUG

    ...
    
    m_PhysicsSystem->OptimizeBroadPhase();
}
```

```cpp
JPH::BodyFilter m_BodyFilter;
JPH::BPLayerInterfaceImpl m_BroadPhaseLayerInterface;  // The broadphase layer interface that maps object layers to broadphase layers
JPH::BroadPhaseLayerFilter m_BroadPhaseLayerFilter;
JPH::ObjectVsBroadPhaseLayerFilterImpl m_ObjectVsBroadPhaseLayerFilter;  // Class that filters object vs broadphase layers
JPH::ObjectLayerPairFilterImpl m_ObjectVsObjectLayerFilter;              // Class that filters object vs object layers
JPH::ObjectLayerFilter m_ObjectLayerFilter;
JPH::PhysicsSystem* m_PhysicsSystem = nullptr;
JPH::BodyInterface* m_BodyInterface = nullptr;
JPH::PhysicsSettings m_PhysicsSettings;
JPH::ShapeFilter m_ShapeFilter;
ContactListenerImp m_ContactListener;

JPH::JobSystem* m_JobSystem = nullptr;
JPH::TempAllocator* m_TempAllocator = nullptr;

JPH::WireframeRenderer* m_WireframeRenderer = nullptr;
```

All of this is done in the main system of this project, called Demo. This is essentially what I needed to get Jolt set up properly. Now obviously, that won't be enough to find out if the physics are actually working properly. For that, I had to make a physics body to test this.

Jolt Physics uses its JPH::Body class for the different physics bodies that the system updates. Since my project uses an ECS ([EnTT](https://github.com/skypjack/entt) to be exact) I had to make my own wrapper class around Jolt's body, otherwise the ECS cannot work with it, because of the complexity of Jolt's class. So I made this:

```cpp
class PhysicsBody
{
public:
    PhysicsBody(bool hasSensor, BodyTypes type);
    ~PhysicsBody();

    const JPH::Body* GetPhysicsBody() const { return m_Body; }
    void SetPhysicsBody(JPH::Body* newBody) { m_Body = newBody; }

    const BodyTypes GetType() const { return m_BodyType; }
    void SetType(BodyTypes newType) { m_BodyType = newType; }

    JPH::Body* GetSensor() { return m_Sensor; }
    void SetSensor(JPH::Body* inSensor) { m_Sensor = inSensor; }

    std::shared_ptr<bee::MeshRenderer> GetMeshRenderer() { return m_MeshRenderer; }
    void SetMeshRenderer(std::shared_ptr<bee::MeshRenderer> renderer) { m_MeshRenderer = renderer; }

    entt::entity GetEntity() { return m_Entity; }
    void SetEntity(entt::entity entity) { m_Entity = entity; }

private:
    JPH::Body* m_Body = nullptr;
    JPH::Body* m_Sensor = nullptr;
    std::shared_ptr<bee::MeshRenderer> m_MeshRenderer = nullptr;
    BodyTypes m_BodyType;

    entt::entity m_Entity;
};
```
Making this wrapper would allow me to create entities with this class as their component. 

Since we are now on the topic of making entities, let's look at how our training dummy is made and added to the scene. 

 ```cpp
 void Demo::CreateTrainingDummy(const Vec3 position)
{
    m_Dummy = Engine.ECS().CreateEntity();
    auto& transform = Engine.ECS().CreateComponent<Transform>(m_Dummy);
    auto& stats = Engine.ECS().CreateComponent<PlayerComponent>(m_Dummy);
    stats.m_DefaultHealth = 500.0f;
    stats.m_Health = stats.m_DefaultHealth;
    stats.m_Speed = 5.0f;

    transform.Name = "Dummy";
    transform.Scale = glm::vec3(1.0f);
    const auto& model = Engine.Resources().Load<Model>("models/CubeAndCylinder.gltf");
    auto& meshRenderer = Engine.ECS().CreateComponent<MeshRenderer>(m_Dummy, model->CreateMeshRendererFromNode("Cylinder"));

    auto& physicsBody = Engine.ECS().CreateComponent<PhysicsBody>(m_Dummy, false, BodyTypes::CHARACTER);
    physicsBody.SetEntity(m_Dummy);

    // Cylinder
    physicsBody.SetPhysicsBody(m_BodyInterface->CreateBody(BodyCreationSettings(
        new CylinderShape(1.0f, 1.0f, 0.0f), RVec3(1 * position), Quat::sIdentity(), EMotionType::Kinematic, Layers::MOVING)));

    if (m_WireframeRenderer != nullptr)
        physicsBody.SetMeshRenderer(m_WireframeRenderer->GetDebugShapeType(JPH::WireframeRenderer::Cylinder));

    transform.Translation = JoltToGlm(physicsBody.GetPhysicsBody()->GetBodyCreationSettings().mPosition);
    // Adds the body to the physics system
    m_BodyInterface->AddBody(physicsBody.GetPhysicsBody()->GetID(), EActivation::DontActivate);

    m_ContactListener.AddBodyToRegistry(physicsBody.GetPhysicsBody()->GetID().GetIndex(), physicsBody.GetEntity());
}
 ```
I first make the entity, and then I create components which are tied to the entity. I make a transform, provided by Bee engine (developed by the teachers at BUAS) and set its scale.

Since we want to also make the dummy a character, with health and all that, we need to give it the PlayerComponent where all of that data is stored, and initialize it's parameters.

We then also make a component for the Physics Body, using the custom class that we made earlier. We then set it by giving it the proper parameters that Jolt has made for their Body class, and a Body Type from the ones that are pre-determined. We also make sure to update the transform to the position of the body since they are 2 sepratae things.


## Creating the Player and the Camera

One of the main parts of this project are the controllable character and its camera. The way we make the player is quite similar to how we make the Dummy, we make an entity to which we give a transform and a PlayerComponent, but instead of the PhysicsBody class, we give it something else - it uses the "CharacterController" class. 

```cpp
class CharacterController
{
public:
	CharacterController() {}
	CharacterController(JPH::PhysicsSystem* physicsSystem);
	~CharacterController();

	void UpdatePlayer(float speed);

	JPH::CharacterVirtual* GetCharacterVirtual() { return m_CharacterVirtual; }
	void SetCharacterVirtual(JPH::CharacterVirtual* newCharacter) { m_CharacterVirtual = newCharacter; }
	JPH::CharacterVirtualSettings* GetSettings() { return m_VirtualSettings; }

	const std::shared_ptr<bee::MeshRenderer> GetMeshRenderer() { return m_MeshRenderer; }
    void SetMeshRenderer(std::shared_ptr<bee::MeshRenderer> renderer) { m_MeshRenderer = renderer; }

	entt::entity GetEntity() { return m_Entity; }
	void SetEntity(entt::entity entity) { m_Entity = entity; }

private:

	entt::entity m_Entity;
	
	JPH::CharacterVirtual* m_CharacterVirtual = nullptr;
	JPH::CharacterVirtualSettings* m_VirtualSettings = nullptr;

	std::shared_ptr<bee::MeshRenderer> m_MeshRenderer = nullptr;
 ```

Again, this is just a wrapper around another one of Jolt's classes, this time for the "CharacterVirtual", the class that the creator of the engine reccomends is used when making a character controller for your game. So we do something like this for the player:

```cpp
void Demo::CreatePlayer(const Vec3 position)
{
    // Character controller
    //create an entity
    //give it a transform

    //initialize a model for the entity

    auto& player = Engine.ECS().CreateComponent<CharacterController>(m_Player, m_PhysicsSystem);

    //give it a PlayerComponent

    player.SetEntity(m_Player);
    player.GetCharacterVirtual()->SetPosition(position);
    
    //initialize debug renderer
    //set transform to the Jolt body position
}
```

The process is very similar to how we create the dummy, but we give the entity the CharacterController class instead.

Now that we have our player, we can set up the camera. Initializing it would look like this:

```cpp
{
    const auto entity = Engine.ECS().CreateEntity();
    auto& transform = Engine.ECS().CreateComponent<Transform>(entity);
    transform.Name = "Camera";
    auto projection = glm::perspective(glm::radians(60.0f), 1.77f, 0.2f, 500.0f);
    auto view = glm::lookAt(glm::vec3(20.0, 0.2f, 8.0f), glm::vec3(0.0f, 0.0f, 1.5f), glm::vec3(0.0f, 1.0f, 0.0f));
    auto& camera = Engine.ECS().CreateComponent<Camera>(entity);
    camera.Projection = projection;
    const auto viewInv = inverse(view);
    Decompose(viewInv, transform.Translation, transform.Scale, transform.Rotation);
}
```
This would initialize the camera, but to actually have it follow the player we need to update it every frame, by doing this:

```cpp
for (auto [entity, camera, transform] : Engine.ECS().Registry.view<Camera, Transform>().each())
{
    offset = transform.Rotation * offset;
    auto view = glm::lookAt(glm::vec3(sin(m_mouseY) * cos(-m_mouseX) * m_radius + playerTransform.Translation.x + offset.x,
        m_radius * cos(m_mouseY) + playerTransform.Translation.y + offset.y,
        sin(m_mouseY) * sin(-m_mouseX) * m_radius + playerTransform.Translation.z + offset.z),
        glm::vec3(playerTransform.Translation.x + offset.x, playerTransform.Translation.y + offset.y, playerTransform.Translation.z + offset.z),
        glm::vec3(0.0f, 1.0f, 0.0f));

    const auto viewInv = inverse(view);
    
    Decompose(viewInv, transform.Translation, transform.Scale, transform.Rotation);
    
    camera.Forward = glm::vec3{ view[0][2], view[1][2], view[2][2] };
    camera.Right = glm::vec3{ view[0][0], view[1][0], view[2][0] };

    camera.Forward = glm::normalize(camera.Forward);
    camera.Right = glm::normalize(camera.Right);

    body.GetCharacterVirtual()->SetRotation(GlmToJolt(glm::normalize(JoltToGlm(Quat(0.0f,transform.Rotation.y, 0.0f , transform.Rotation.w)))));
}
```

To make the camera follow the player, I am using an implementation for the camera from one of the other example projects that come with the "Bee" engine ("model_viewer.cpp" to be precise). I first calculate our view matrix, making sure to set the center of the camera as the player's poition in addition to a small offset since I want the camera to be slightly to the right of the player.

Then, after constructing the view we are able to extract the Forward and Right vectors of the camera. The `glm::lookAt` function contains those vectors, we just need to extract them from the matrix. 

![view matrix](/images/viewMatrix.png)

As seen from the picture from the [LearnOpenGL](https://learnopengl.com/Getting-started/Camera) website, the Forward and Right vectors are in the first and third row of the matrix. After extracting them I make sure to normalize them since they will be used for calculating the movement of the player.


```cpp
void CharacterController::UpdatePlayer(float speed)
{
   ...

    glm::vec3 velocity{};
    velocity += (left ? -newCamera.Right : glm::vec3(0.0f)) +
                (right ? newCamera.Right : glm::vec3(0.0f)) +
                (forward ? -newCamera.Forward : glm::vec3(0.0f)) +
                (backwards ? newCamera.Forward : glm::vec3(0.0f));

    velocity.y = 0.0f;

    if (glm::length(velocity) > 0.0f)
    {
        velocity = glm::normalize(velocity);
    }

    newVelocity.SetX(velocity.x * speed);
    newVelocity.SetZ(velocity.z * speed);

    if (m_CharacterVirtual != nullptr) m_CharacterVirtual->SetLinearVelocity(newVelocity);
}
```
I calculate the velocity of the player depending on which direction they need to go. I then normalize the velocity vector, and multiply it by the walking speed of the player and I call `SetLinearVelocity` to update the player's new velocity.

After all this, I ended up with the following:

<video width="500" height="500" controls>
  <source src="/videos/cameraShowcase.mp4" type="video/mp4">
</video>

You might notice that there is a crosshair in the middle of the screen. That crosshair is an entity with a mesh and a texture. The only thing that is different is that when I create its transform, I set its parent to be the transform of the camrea. That means that the position of the crosshair will be relative to that of the camera
```cpp
void Demo::CreateCrosshair(float size, const std::string& path)
{
    ...
    auto& transform = Engine.ECS().CreateComponent<Transform>(entity);
    transform.SetParent(view.front());
    transform.Name = "Crosshair";
    transform.Scale = glm::vec3(1.0, 0.0f, 1.0);
    ...
}
```
The crosshair will come into play in the following section.



## The ability system and the abilities

The technical design of the system drew some inspiration from Unreal's Gameplay Ability System, mainly for it's data driven approach.

That said, let's look at what the ability in this project looks like:

```cpp
struct AbilityAttributes
{
    int m_Damage = 0;
    float m_Cooldown = 0.0f;
    float m_CooldownTimer = 0.0f;
    float m_Size = 0.0f;

    // Will be used for the projectile
    float m_Speed = 0.0f;
    // could be implemented in the form of a timerS
    float m_Lifetime = 0.0f;
    float m_MaxLifetime = 0.0f;

    // Will be used exclusively for the "Blink ability"
    float m_Duration = 0.0f;

    bool m_Cast = false;
    bool m_ToBeDestroyed = false;
    bool m_HasCollided = false;

    bool m_CanStun = false;
    float m_StunDuration = 0.0f;

    AbilityType m_Type{};

    std::string m_Name = "Name";

    bee::Input::KeyboardKey m_Keybind;
    ...
}
```

The ability is just a simple struct with different variables that can be changed. This struct is used as a component to an entity, which can also have a physics body attached to it (in the case of the AOE and Projectile in this project). This struct is also part of an unordered map that the Player Component has, where the different abilities, that the players have, are stored.

One of the variables inside the struct is the `m_Keybind`. When a new ability is created, this keybind is automatically set to a key on the keyboard, depending on the type of ability. At the moment, there is no easy way to give different inputs to the abilities, but is something that will be added in the future.

Given the keybinds, the way that the abilities are cast works as follows:

```cpp
void AbilitySystem::HandleAbilities(float dt)
{
    for (auto [entity, transform, player] : Engine.ECS().Registry.view<Transform, PlayerComponent>().each())
    {
        for (auto& [abilityName, attributes] : player.m_Abilities)
        {
            if (attributes.m_CooldownTimer <= 0.0f && Engine.Input().GetKeyboardKeyOnce(attributes.m_Keybind))
            {
                CastAbility(transform, attributes, player);
            }
            else if (attributes.m_CooldownTimer > 0.0f)
            {
                attributes.m_CooldownTimer -= dt;
            }

        }
    }
    UpdateAbilityDamage();
    HandleAbilityLifetime(dt);
}
```

The ability system loops over the different abilities that the players have and checks if there is input for the corresponding ability. Then, the follwoing function is called:

```cpp
void AbilitySystem::CastAbility(const bee::Transform& inTransform, AbilityAttributes& attributes, PlayerComponent& player)
{
    if (attributes.m_Type == AbilityType::PROJECTILE)
    {
        CastProjectile(inTransform, attributes);
    }

    else if (attributes.m_Type == AbilityType::AOE)
    {
        CastAOE(inTransform, attributes);
    }

    else if (attributes.m_Type == AbilityType::BLINK)
    {
        CastBlink(attributes, player);
    }
}
```

It checks what type of ability needs to be executed and calls the appropriate function for that. To create any of those abilities, there are 3 main functions that need to be used:

```cpp
// Encapsulated logic for creating an ability entity
entt::entity SetupAbilityEntity(AbilityAttributes& attributes);
void SetupPhysicsBody(entt::entity entity, const bee::Transform& playerTransform, const JPH::Shape* shape,
                      glm::vec3 offset, JPH::EMotionType motionType, JPH::WireframeRenderer::WireframeTypes type, BodyTypes bodyType);
void SetupAbilityTransform(entt::entity entity, glm::vec3 scale);
```

This sets up most of the things that an ability would need. It's a bit different for the projectile, if you want it to work with the crosshair aiming in the project, you'd have to also call the `RayCast` and `SetupProjectileVelocity` functions. The blink on the other hand requires very little, you just need to call `SetUpAbilityEntity` and then set the player's speed to the `m_Speed` attribute of the ability. The different examples are shown below.

### Projectile

We'll start off by looking at the projectile ability:

```cpp
void AbilitySystem::CastProjectile(const Transform& playerTransform, AbilityAttributes& attributes)
{
    auto projectile = SetupAbilityEntity(attributes);
    glm::vec3 hitPos = RayCast();
    SetupAbilityTransform(projectile, glm::vec3(attributes.m_Size));

    auto& transform = Engine.ECS().Registry.get<Transform>(projectile);
    const auto& model = Engine.Resources().Load<Model>("models/sphere.gltf");
    auto& meshRenderer = Engine.ECS().CreateComponent<MeshRenderer>(projectile, model->CreateMeshRendererFromNode("Mesh"));

    SetupPhysicsBody(projectile, playerTransform, new SphereShape(transform.Scale.x),
        glm::vec3{ 0.0, transform.Scale.y, -2.0f - transform.Scale.x }, EMotionType::Kinematic,
        JPH::WireframeRenderer::WireframeTypes::Sphere, BodyTypes::PROJECTILE);
    SetupProjectileVelocity(projectile, hitPos);
    attributes.m_CooldownTimer = attributes.m_Cooldown;
}
```

As I mentioned earlier, since shooting projectiles in this project is meant to work with crosshair aiming, we need to use the `RayCast` function and the `SetupProjectileVelocity`. 

```cpp
    void SetupProjectileVelocity(entt::entity entity, glm::vec3 hitPos);
    glm::vec3 RayCast();
```

The `RayCast` returns a `glm::vec3` which is can be used as the `hitPos` argument inside the function wehere we set the projectile velocity. As things stand, the ray cast works exclusively with the camera of the controllable player, so making it work with the dummy, for example, would give a bit weird results.

Below you can see a video that showcases the projectile ability in action:

<video width="640" height="500" controls>
  <source src="/videos/projectileShowcase.mp4" type="video/mp4">
</video>

## AOE

The second type of ability in this system is the Area of Effect, AOE for short. 

To make an AOE, the procedure is similar to the one of making a projectile, without having the ray cast part in it:

```cpp
void AbilitySystem::CastAOE(const Transform& inTransform, AbilityAttributes& attributes)
{
    auto aoe = SetupAbilityEntity(attributes);
    SetupAbilityTransform(aoe, glm::vec3{ attributes.m_Size, 0.1, attributes.m_Size });

    auto& transform = Engine.ECS().Registry.get<Transform>(aoe);
    const auto& model = Engine.Resources().Load<Model>("models/CubeAndCylinder.gltf");
    auto& meshRenderer = Engine.ECS().CreateComponent<MeshRenderer>(aoe, model->CreateMeshRendererFromNode("Cylinder"));

    RefConst<CylinderShape> cylinder = new CylinderShape(1.0f, 1.0f);
    SetupPhysicsBody(aoe, inTransform, new ScaledShape(cylinder, GlmToJolt(transform.Scale)),
        glm::vec3{ 0.0f, -0.9f, -20.0f - transform.Scale.z }, EMotionType::Static,
        JPH::WireframeRenderer::Cylinder, BodyTypes::AOE);

    attributes.m_CooldownTimer = attributes.m_Cooldown;
}
```
After doing this, here is what we get:

<video width="640" height="500" controls>
  <source src="/videos/aoeShowcase.mp4" type="video/mp4">
</video>

## Blink/Dash

The way the blink works is a lot simpler. It requirse just an entity, and inside of the function where the abiltiy is cast, we need to set the speed of the player to the `m_Speed` attribute of the ability. As things stand, this ability turned out to be more like a dash than a blink/teleport, but making this in the future would not require that much work.

```cpp
void AbilitySystem::CastBlink(AbilityAttributes& newAttributes, PlayerComponent& player)
{
    auto blink = Engine.ECS().CreateEntity();
    auto& attributes = Engine.ECS().CreateComponent<AbilityAttributes>(blink);

    attributes = newAttributes;
    attributes.m_Cast = true;
    attributes.m_Name = "Blink";
    player.m_Speed = attributes.m_Speed;

    newAttributes.m_CooldownTimer = newAttributes.m_Cooldown;
}
```

<video width="640" height="500" controls>
  <source src="/videos//blinkShowcase.mp4" type="video/mp4">
</video>

## Dealing damage and applying status effects

We went through how the different abilities are made and how they are cast, but we need to also go through how exactly they interact with the world. The way they do that is through something called the ContactListener:

```cpp
void ContactListenerImp::OnContactAdded(const JPH::Body& inBody1, const JPH::Body& inBody2,
                                        const JPH::ContactManifold& inManifold, JPH::ContactSettings& ioSettings)
{
    auto type1 = GetBodyType(inBody1.GetID().GetIndex());
    auto type2 = GetBodyType(inBody2.GetID().GetIndex());   

    auto& body1 = Engine.ECS().Registry.get<PhysicsBody>(type1);
    auto& body2 = Engine.ECS().Registry.get<PhysicsBody>(type2);

    if (body1.GetType() == BodyTypes::CHARACTER && body2.GetType() == BodyTypes::PROJECTILE)
    {
        std::cout << "Character hit by projectile" << "\n";

        auto& attributes = Engine.ECS().Registry.get<AbilityAttributes>(type2);
        auto& stats = Engine.ECS().Registry.get<PlayerComponent>(type1);

        stats.m_IsHit = true;
        if(attributes.m_CanStun)
        {
            stats.m_IsStunned = true;
            stats.m_stunDuration = attributes.m_StunDuration;
        }
        attributes.m_HasCollided = true;
        attributes.m_ToBeDestroyed = true;

    }
    else if (body1.GetType() == BodyTypes::CHARACTER && body2.GetType() == BodyTypes::AOE)
    {
        std::cout << "Character hit by AOE" << "\n";

        auto& attributes = Engine.ECS().Registry.get<AbilityAttributes>(type2);
        auto& stats = Engine.ECS().Registry.get<PlayerComponent>(type1);

        stats.m_IsHit = true;

        if(attributes.m_CanStun)
        {
            attributes.m_HasCollided = true;
            stats.m_IsStunned = true;
            stats.m_stunDuration = attributes.m_StunDuration;
        }
    }
    else if (body1.GetType() == BodyTypes::TERRAIN && body2.GetType() == BodyTypes::PROJECTILE)
    {
        std::cout << "Terrain hit by projectile" << "\n";

        auto& attributes = Engine.ECS().Registry.get<AbilityAttributes>(type2);
        attributes.m_HasCollided = true;
    }
}
```

This is the function that is executed when one of the abilities collides with something in the world. I made an unordered map, inside of which I am adding all the physics bodies upon their creation. The bodies have different types (PROJECTILE, AOE, TERRAIN and CHARACTER). Depending on what type of body we have, the collision resolution will be different. 

The first thing we do here is we get the entities of the two colliding bodies. Since the unordered map has the Body ID as the key and the PhysicsBody as the value, we can easily get the body out of the map, and get it's entity, which stored inside it. Then, the checks are made to see what types of bodies are collidng, and the corresponding collision gets resolved. 

When an ability collides with a player, we set the player's flag `m_IsHit` to true, and update their health in the Ability system:

```cpp
void AbilitySystem::UpdateAbilityDamage()
{
    for (auto [ability, attributes] : Engine.ECS().Registry.view<AbilityAttributes>().each())
    {
        for (auto [entity, body, stats] : Engine.ECS().Registry.view<PhysicsBody, PlayerComponent>().each())
        {
            if(stats.m_IsHit)
            {
                stats.m_Health -= attributes.m_Damage;
                if (stats.m_Health < 0.0f)
                {
                    stats.m_Health = 0.0f;
                }
            }
            stats.m_IsHit = false;
        }
    }
}
```
As for the status effects, at the moment they are applied in the other system, Demo, as follows:

```cpp
//Applying a stun effect on the player
for (auto [entity, body, stats] : Engine.ECS().Registry.view<PhysicsBody, PlayerComponent>().each())
{
    if (stats.m_IsStunned)
    {
        stats.m_stunDuration -= dt;
        stats.m_Speed = 0.0f;
        if (stats.m_stunDuration < 0.0f)
        {
            stats.m_IsStunned = false;
            stats.m_Speed = stats.m_DefaultSpeed;
        }
    }
}
```

Below is a video where you can see all of this happen:

<video width="640" height="500" controls>
  <source src="/videos/damageAndStunShowcase.mp4" type="video/mp4">
</video>

## Ability creation and loading

We looke through the different abilities, how we make them, how we use them etc. Now it's time to see how we can add new abilities and load them to the different player in the world.

Let's now look at how ability creation works:

<video width="640" height="500" controls>
  <source src="/videos/createAbilityShowcase.mp4" type="video/mp4">
</video>

The user can name their ability whatever they want. Then, they can give the ability whatever properties they like, as long as they are valid. If not, the console will print out the error that the user has made. When clicking the "Save Ability" button, a ".JSON" file gets created, and allows the system to reuse the abiltiy. The newly saved ability gets added to the chosen players list of abilities. 

Abilities can also be loaded and then used by whatever player owns the ability. The video below showcases how it works:


<video width="640" height="500" controls>
  <source src="/videos/loadAbiltiyShowcase.mp4" type="video/mp4">
</video>

## What's next for the project?

As things stand, the project has a pretty solid base for an ability system. There is a nice selection of abilities that the player can use and also the ability to save and load them. The next step would be to make parts of the system more modular so that they allow easier management of new ability parameters (new status effects e.g.). There is also the need of a better way to give the abilities keybinds, so some sort of an Input System is part of the plan. Apart from that, I'll have to see how the system changes over time so that I can adapt it to the new things I'd like to add to it.

## Conclusion

Having worked on this project for the past 2 months was quite challenging but also fun. I learned a lot about working with an external physics library, technical design of an ability system and much more. I learned new skills, both coding skills and organizational skills, which will be very valuable as I continue my journey of becmoing a game developer. I will most likely continue working on this project in my free time, because it has the potential to be something much, much better than it is at the moment. 

![buas logo](/images/buasLogo.png)

<!-- ```cpp
void Demo::CreatePlayer(const Vec3 position)
```

This is how I am creating my player. It looks a bit more different than making the training dummy, since a character controller has a different physics body class.

```cpp
void Demo::CreateTrainingDummy(const Vec3 position)
```

For the dummy, I need to initialize more things since it uses my own PhyscisBody class which used as a wrapper around Jolt's Body class. Because it's a Jolt Body, I also need to add it to a registry that I made later on for collision resoltuion since Kinematic bodies only have collision detection, contacts are not resolved automatically.

<video width="500" height="500" controls>
  <source src="/videos/character_controller.mp4" type="video/mp4">
</video>


## Milestone 2
### The player can use the 3 different ability types (projectile, AOE, blink). ImGui integration for easier tweaking of character and ability parameters. 

In the project, the player has an unordered map of abilities. The ability is a simple struct with variables.

```cpp
struct AbilityAttributes
{
    int m_Damage = 0;
    float m_Cooldown = 0.0f;
    float m_CooldownTimer = 0.0f;
    float m_Size = 0.0f;

    // Will be used for the projectile
    float m_Speed = 0.0f;
    // could be implemented in the form of a timerS
    float m_Lifetime = 0.0f;
    float m_MaxLifetime = 0.0f;

    // Will be used exclusively for the "Blink ability"
    float m_Duration = 0.0f;

    bool m_Cast = false;
    bool m_ToBeDestroyed = false;
    bool m_HasCollided = false;

    bool m_CanStun = false;
    float m_StunDuration = 0.0f;

    AbilityType m_Type{};

    std::string m_Name = "Name";

    bee::Input::KeyboardKey m_Keybind;
    ...
}
```
One of them is a keybind which the ability is assigned upon creation, dpending on its type (Projectile, AOE or Blink). Then, in the Update of the ability system, I loop over the players abilities, and check is a key is pressed that matches the value of the "m_Keybind" variable of the ability.


Then, CastAbility is called, which takes the type of the ability, and calls the cast function of the appropriate ability:

```cpp
void AbilitySystem::CastAbility(const bee::Transform& inTransform, AbilityAttributes& attributes, PlayerComponent& player)
{
    if (attributes.m_Type == AbilityType::PROJECTILE)
    {
        CastProjectile(inTransform, attributes);
    }

    else if (attributes.m_Type == AbilityType::AOE)
    {
        CastAOE(inTransform, attributes);
    }

    else if (attributes.m_Type == AbilityType::BLINK)
    {
        CastBlink(attributes, player);
    }
}
```

Casting any of those abilities calls the main fucntions used for making an ability which are:
```cpp
// Encapsulated logic for creating an ability entity
entt::entity SetupAbilityEntity(AbilityAttributes& attributes);
void SetupPhysicsBody(entt::entity entity, const bee::Transform& playerTransform, const JPH::Shape* shape,
                      glm::vec3 offset, JPH::EMotionType motionType, JPH::WireframeRenderer::WireframeTypes type, BodyTypes bodyType);
void SetupAbilityTransform(entt::entity entity, glm::vec3 scale);
```
This sets up most of the needed information for the abilities. If you want to have a projectile and make it work with the crosshair aiming in the project, you'd have to also call the "RayCast" and "SetupProjectileVelocity" functions. 


In the video below, you could see how I can set the health of the training dummy to a value I'd like or change its movement speed. 

<video width="640" height="500" controls>
  <source src="../assets/media/characterParameters.mp4" type="video/mp4">
</video>

## Projectile 
A setup for casting a projectile would look like this:

```cpp
void AbilitySystem::CastProjectile(const Transform& playerTransform, AbilityAttributes& attributes)
{
    auto projectile = SetupAbilityEntity(attributes);
    glm::vec3 hitPos = RayCast();
    SetupAbilityTransform(projectile, glm::vec3(attributes.m_Size));

    auto& transform = Engine.ECS().Registry.get<Transform>(projectile);
    const auto& model = Engine.Resources().Load<Model>("models/sphere.gltf");
    auto& meshRenderer = Engine.ECS().CreateComponent<MeshRenderer>(projectile, model->CreateMeshRendererFromNode("Mesh"));

    SetupPhysicsBody(projectile, playerTransform, new SphereShape(transform.Scale.x),
        glm::vec3{ 0.0, transform.Scale.y, -2.0f - transform.Scale.x }, EMotionType::Kinematic,
        JPH::WireframeRenderer::WireframeTypes::Sphere, BodyTypes::PROJECTILE);
    SetupProjectileVelocity(projectile, hitPos);
    attributes.m_CooldownTimer = attributes.m_Cooldown;
}
```

Since shooting in this project is built around the crosshair aiming, you need to add 2 additional functions so that you can make a projectile:

```cpp
    void SetupProjectileVelocity(entt::entity entity, glm::vec3 hitPos);
    glm::vec3 RayCast();
```
Using the vector that you get from the RayCast as the "hitPos" argument, you'd get the result that you need. 
(Note: the ray cast works only with the camera that is attached to the player in the project, so making it work with a different player would require changes to the camera).

And below is what the projectile would look like in the project:

<video width="640" height="500" controls>
  <source src="../assets/media/projectileShowcase.mp4" type="video/mp4">
</video>

Using the ray cast from the Jolt Physcis Enigne, the player is able to shoot a projectile to where the crosshair is aiming at. 

## AOE

For the AOE, you can essentially skip the RayCast and velocity part and just use the other functions to setup your AOE ability. Once you do that, you will end up with something like this:

```cpp
void AbilitySystem::CastAOE(const Transform& inTransform, AbilityAttributes& attributes)
{
    auto aoe = SetupAbilityEntity(attributes);
    SetupAbilityTransform(aoe, glm::vec3{ attributes.m_Size, 0.1, attributes.m_Size });

    auto& transform = Engine.ECS().Registry.get<Transform>(aoe);
    const auto& model = Engine.Resources().Load<Model>("models/CubeAndCylinder.gltf");
    auto& meshRenderer = Engine.ECS().CreateComponent<MeshRenderer>(aoe, model->CreateMeshRendererFromNode("Cylinder"));

    RefConst<CylinderShape> cylinder = new CylinderShape(1.0f, 1.0f);
    SetupPhysicsBody(aoe, inTransform, new ScaledShape(cylinder, GlmToJolt(transform.Scale)),
        glm::vec3{ 0.0f, -0.9f, -20.0f - transform.Scale.z }, EMotionType::Static,
        JPH::WireframeRenderer::Cylinder, BodyTypes::AOE);

    attributes.m_CooldownTimer = attributes.m_Cooldown;
}
```
<video width="640" height="500" controls>
  <source src="/videos/aoeShowcase.mp4" type="video/mp4">
</video>

The player can now use the AOE ability. It's casted a bit in front of the player, dealing damage if it hits an enemy (if the ability has an amount of damage set to it).

## Blink

The way the blink is a lot simpler. Since it's just changing the speed of the player, it does not require a body or a transform, so what you need to use to make a blink/dash would be just this:

```cpp
void AbilitySystem::CastBlink(AbilityAttributes& newAttributes, PlayerComponent& player)
{
    auto blink = Engine.ECS().CreateEntity();
    auto& attributes = Engine.ECS().CreateComponent<AbilityAttributes>(blink);

    attributes = newAttributes;
    attributes.m_Cast = true;
    attributes.m_Name = "Blink";
    player.m_Speed = attributes.m_Speed;

    newAttributes.m_CooldownTimer = newAttributes.m_Cooldown;
}
```

<video width="640" height="500" controls>
  <source src="/videos/blinkShowcase.mp4" type="video/mp4">
</video>

The player can now use the blink/dash ability. Upon a button press, the player blinks/dashes in the direction they are mvoing.


## Milestone 3
### Add functionality to create new abilities using ImGui. Make a system that can save the ability, using the different parameters (range, radius, cooldown, etc. ) that the user has given as input, using serialization. Add a third person camera and collision resolution for abilities. Make it more modular so that the user can add new abilities from the ones that are already preset in the program. 

Using the "Create Ability" menu, the user can add a new ability and give it to the players that are currently in the world. The user can input any parameters they want, and upon clicking the "Save Ability", it will be added to the player's unordered map of abilities.

<video width="640" height="500" controls>
  <source src="/videos/createAbilityShowcase.mp4" type="video/mp4">
</video>

Creating a new ability and adding it to the player. The user has the freedom to give the ability whatever parameters they like. When clicking the "Save Ability" button, a ".JSON" file gets created, and allows the system to reuse the abiltiy. The newly saved ability gets added to the chosen players list of abilities. 

<video width="640" height="500" controls>
  <source src="/videos/loadAbiltiyShowcase.mp4" type="video/mp4">
</video>

Preset abilities can also be loaded from the menu shown in the video above, and then used by the player.

<video width="640" height="500" controls>
  <source src="/videos/cameraShowcase.mp4" type="video/mp4">
</video>


Apart from saving and loading the new abilities using ImGui, I also added the third person camera, as already seen in the other videos, and the crosshair used for aiming. Collision resolution for abilities is also working, as seen from the video below.

<video width="640" height="500" controls>
  <source src="/videos/collisionResolution.mp4" type="video/mp4">
</video>

![buas logo](/images/buasLogo.png) -->