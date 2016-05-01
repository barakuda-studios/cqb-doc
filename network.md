# CQB Network

## Summary

* [Basic network explanation](#basic-network-explanation)
* [The Photon views](#the-photon-views)
* [The RPCs ( Remote Procedure Call )](#the-rpcs)

## Basic network explanation

In CQB, we use the network plugin " Photon Networking ", provided by exitgames. This plugin could allow us to run our own server ( as the Unity plugin ), but we use it because it can also provide servers hosted by exitgames ( so we don't have to run our own servers ). We're on the free plan, so we can have a limit of 20 CCU ( concurrent users – players at the same time ), but if the game has a little success I'll maybe upgrade it to the 95$ plan to have 100 CCU.

As the servers are hosted by exitgames, we don't have to do all the code on the server side, so we just have to code the players side. Basically it means that we don't need to code how the server will control the matches and the reliability of the values we send to him, so a big downside is that he relies completely on what the clients send to him ( so if a hacker want to do 1 000 000 damages the server can't control it to realise it's impossible ).

For the network coding, we don't go to the server side and we only code the clients side. So basically, the network system you have to keep in mind while coding is not " server – clients " but " masterclient – clients ". The server is only there to send the variables / functions from client to client.

## The Photon views

The network views are common to all the multiplayer games. Here, we use photonView because we use Photon Networking ( with the built-in networking engine from Unity we use networkView ).

So basically the photonView is the view of the network that each player has.

Each player has its own view, with its own variables, and that gives problems when transmitting variables that are global in the network: It's the problem of the synchronization.

Here is one example of the networks synchronization  I actually had to deal with:

Take a variable from the network manager ( let's call it `playerCount`, its value at the start is 0 ).


When the first player joins it becomes the masterclient, and takes the value of `playerCount=0`;

Each player has a function which adds +1 to playerCount whenever a new player joins ( the variable is used to count the players ). So as the masterclient has joined, it adds +1 to `playerCount`.

Then 8 players join, so the masterclient now has a value of 1+8=9 for `playerCount`.

Let's say a last player joins. The master client will add +1, and the new player will do this too. So the masterclient will have 10, but as the last client just joined, it only has 1.

In fact, when the last client joined he saw this:

Take value of `playerCount=0`;
If a player joins, add +1.
I joined so I add +1.
playerCount=1.

So he did it right, he added +1 to the value, but he only counted him because the function tells him to count the new players. He's not aware that the value for the masterclient is 10.


You surely didn't understand this well but let's imagine another example, and it's actually a true example as it's a bug in the game I still didn't fixed ( because there are a lot of things to fix because of the network )

In our network manager script we have 2 values: redCount and blueCount, each one counting respectively the number of red players and blue players.

So whenever a new player joins we add them to a team and add +1 to either redCount or blueCount.

So this is actually EXACTLY like in the example, if we don't synchronize the values over the network, each player will have different numbers, so the last player who joined won't be aware of how many players there actually are.

Keep that problem in mind because whenever you need to do a network function you will need this a lot. Plus we'll soon begin to do the deathmatch / team deathmatch system, with a score system and scoreboards displaying the score of each player. So if you don't understand the problem of the synchronization of values over the network it will be hard.

So to fix this problem we have 2 solutions differing from the type of values:

For the values which differ depending on the player, we put them in the function " OnPhotonSerializeView ". This function, which you can see in the script "playerNetwork" is used to keep the value of each player updated.

His basic use is to keep the position and rotation of each player updated over each other player view so that everyone can see everyone moving. Without that you wouldn't see the other players as you couldn't know their positions.

Though, we don't use that often as it's main use is for the player's position and rotation.

For the global value, we'll use RPCs to send them to the other players ( the definition of a RPC is given in the part III )

So basically the player who has the correct values ( it's mainly the masterclient ) will send these values to the other players so they don't have wrong values.

He can send them every frame but it's not optimized and it will eat network connection so we need to make functions which will just resend them if they change ( so instead of sending them every time we just update the, it's a big part of the network optimization )


## The RPCs

A RPC is, as its name says, a Remote Procedure Call. It is used to run a function on another player.

A basic example: if you shoot an ennemy how do the masterclient says to a player in which team he is ?

You can't use the "GetComponent()" function to modify its value because the value of the team of the player is in his photonview and not yours ( after all every player has the value of its own team )

That's where you use a RPC.

RPCs are sent like these: photonView.RPC("function", target, optionnal parameter )
You can only send these types of parameters trhough RPCs: int; float; string; PhotonPlayer; PhotonViewID; Vector3; Quaternion

You can put the RPC attribute on function like this:

```cs
[RPC]
void setTeam(string team)
{
	playerTeam=team;
}
```

Then, if you want to know in which team you are, you just have to request the masterclient to send you a RPC ( remember the variables synchronization, the masterclient always have the right values even if the others are updated ).

So you create another RPC ( in which you send your player ID ):

( To get your playerID, do this:

PhotonView myView = PhotonView.Get(this); // Get your photonview
int playerID = myView.owner.ID; //Get the ID of the owner of your photonview (you)

```cs
[RPC]
void askTeam(int playerID)
{
	PhotonPlayer player = PhotonPlayer.Find(playerId);
	/* The masterclient get your player from your playerID so he can send the answer to you only */

	photonView.RPC("setTeam", player, "Blue");
	/* He sends you the Rpc to attribute you a team */

}
```

And to ask the masterclient, you just send him the askTeam RPC with you playerID:

photonView.RPC("askTeam", PhotonTargets.MasterClient, myPlayerID);

He will then execute the askTeam function, which will make him send you the setTeam RPC

The PhotonTargets.target permits you to target group of people instead of getting their playerId and all.

For example you have: PhotonTargets.All, PhotonTargets.Others, PhotonTargets.MasterClient, PhotonTargets.AllBuffered...

If you look into the script playerSpawn.cs, on the playerCreator prefab you will see that this is actually how the masterclient attributes teams.




**BIG EXAMPLE OF HOW USING RPC ( WITH NO CODE )**

In a very close future we will need to make a big use of photonViews and RPCs.

It will be for the scoreboards.

We will take the process of updating the scoreboard when you kill an ennemy.
Here is how it will work:

When you kill the enemy, you need to update your score to add +1 kill to your killCount, so you need to send a RPC to all the other players saying them to add +1 to your killCount.

The enemy you just kill has to update his deathCount, so he just send a RPC to all the others player saying " add +1 to my deathCount "

You then need to update the score of your team.

Remember: for the values of the variables, you should always get those of the masterclient. So you send a RPC to the masterclient saying him " add +1 to the score of my team "

But then the score of all the other players is outdated.

So the masterclient needs to send a RPC to all the players saying them " add +1 to this team "
