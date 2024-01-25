## Making an Ability System for a Third Person Character Controller

## Milestone 1
### Setting up Jolt physics and making a moveable character, and a training dummy 

The main focus during the first week was to make Jolt work and add a controllabel character, and a training dummy. 

```cpp
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
  <source src="../assets/media/createAbilityShowcase.mp4" type="video/mp4">
</video>

Creating a new ability and adding it to the player. The user has the freedom to give the ability whatever parameters they like. When clicking the "Save Ability" button, a ".JSON" file gets created, and allows the system to reuse the abiltiy. The newly saved ability gets added to the chosen players list of abilities. 

<video width="640" height="500" controls>
  <source src="../assets/media/loadAbiltiyShowcase.mp4" type="video/mp4">
</video>

Preset abilities can also be loaded from the menu shown in the video above, and then used by the player.

<video width="640" height="500" controls>
  <source src="../assets/media/cameraShowcase.mp4" type="video/mp4">
</video>


Apart from saving and loading the new abilities using ImGui, I also added the third person camera, as already seen in the other videos, and the crosshair used for aiming. Collision resolution for abilities is also working, as seen from the video below.

<video width="640" height="500" controls>
  <source src="../assets/media/collisionResolution.mp4" type="video/mp4">
</video>

- Explain why I used Jolt, how it's crucial to the project