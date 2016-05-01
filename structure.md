I just got back on the project and don't even understand the structure myself, just have to remember it a little bit

# Scripts

## Weapons

### bulletScript ~~*deprecated*~~

Script used when bullets were RigidBody components, now bullets are just rays sent from the cannon, so everything handling shooting and hitting shit is contained inside `shotScript.cs`

### globalWeaponStats

Contains the variables used to control the weapons, such as recoil, damage, weapon name, mag size, etc. Variables are stored in arrays

### shotScript

Script controlling the shooting and the verification of hits ( verify if you hit a player )
