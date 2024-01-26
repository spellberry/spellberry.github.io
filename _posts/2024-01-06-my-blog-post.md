## Ability System for a Third Person Character Controller

Developing an ability system has been something that I've wanted to do since I've started to get more and more comfortable with programming. In the past 8 weeks, I've been developing an ability system for a game with a third person character controller. Below you will see a short shwocase video of what the program offers:
- A simple third-person character controller with a directional camera and a crosshair for aiming.
- Three types of abilites that the demo character can use - Projectile, AOE and Blink/Dash  
- The ability to tweak parameters to your liking for the different abilities
- You can also create a new ability and give it to one of the characters in the world (you can also load in whatever abilities are already saved to any of the players)

<video width="500" height="500" controls>
  <source src="/videos/generalShowcase.mp4" type="video/mp4">
</video>

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