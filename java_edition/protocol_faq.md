# 协议FAQ

People very, very often have questions regarding the Minecraft Modern Protocol, so we'll try to address some of the most common ones on this document. If you're still having trouble, join us on IRC, channel #mcdevs on irc.libera.chat.

## Is the protocol documentation complete?

Depending on your definition, yes! All packet types are known and their layout documented. Some finer details are missing, but everything you need to make functional programs is present. We also collect information on the pre-release protocol changes, allowing us to quickly document new releases.

## What's the normal login sequence for a client?

See Authentication for communication with Mojang's servers.

The recommended login sequence looks like this, where C is the client and S is the server:

1. Client connects to the server
2. C→S: Handshake State=2
3. C→S: Login Start
4. S→C: Encryption Request
5. Client auth
6. C→S: Encryption Response
7. Server auth, both enable encryption
8. S → C: Set Compression (Optional, enables compression)
9. S → C: Login Success
10. S → C: Login (play)
11. S → C: Plugin Message: minecraft:brand with the server's brand (Optional)
12. S → C: Change Difficulty (Optional)
13. S → C: Player Abilities (Optional)
14. C → S: Plugin Message: minecraft:brand with the client's brand (Optional)
15. C → S: Client Information
16. S → C: Set Held Item
17. S → C: Update Recipes
18. S → C: Update Tags
19. S → C: Entity Event (for the OP permission level; see Entity statuses#Player)
20. S → C: Commands
21. S → C: Recipe
22. S → C: Player Position
23. S → C: Player Info (Add Player action)
24. S → C: Player Info (Update latency action)
25. S → C: Set Center Chunk
26. S → C: Light Update (One sent for each chunk in a square centered on the player's position)
27. S → C: Chunk Data and Update Light (One sent for each chunk in a square centered on the player's position)
28. S → C: Initialize World Border (Once the world is finished loading)
29. S → C: Set Default Spawn Position (“home” spawn, not where the client will spawn on login)
30. S → C: Synchronize Player Position (Required, tells the client they're ready to spawn)
31. C → S: Confirm Teleportation
32. C → S: Set Player Position and Rotation (to confirm the spawn position)
33. C → S: Client Command (sent either before or while receiving chunks, further testing needed, server handles correctly if not sent)
34. S → C: inventory, entities, etc

# What does the normal status ping sequence look like?

When a Notchian client and server exchange information in a status ping, the exchange of packets will be as follows:

1. C → S: Handshake with Next State set to 1 (Status)
2. Client and Server set protocol state to Status.
3. C → S: Status Request
4. S → C: Status Response
5. C → S: Ping Request
6. S → C: Pong Response
7. S → ❌: Server terminates connection to client

(Note that C is the Notchian client and S is the Notchian server).

# Offline mode

If the server is in offline mode, it will not send the Encryption Request packet, and likewise, the client should not send Encryption Response. In this case, encryption is never enabled, and no authentication is performed.

Clients can tell that a server is in offline mode if the server sends a Login Success without sending Encryption Request.

# I think I've done everything right, but…

## …my player isn't spawning!

The Minecraft client will wait at the "Loading Terrain..." screen until late in the login sequence. At the time of writing (version 1.19.3), the server must send the Set Default Spawn Position before the client will spawn the player. In past versions, you could send the player position packet. In general, try sending packets that inform the client about the player's position in the world in order to get past the loading terrain screen.

As of 1.19.3, the minimum packets that need to be received and sent by the server in order to get the client past the loading terrain screen and into the world appear to be:

1. Receive Handshake
2. Receive Login Start
3. Send Login Success
4. Send Login (Play)
5. Send Set Default Spawn Position

The most difficult part of this may be sending any necessary NBTs in the Login (Play) packet. You will probably need to record one from the standard server and replay it. Or you can find someone who has done that already, for example in the PrismarineJS/minecraft-data repo on GitHub. Then you'll need to transform the JSON representation to binary NBT.

## …my client isn't receiving complete map chunks!

Main article: How to Write a Client

The standard Minecraft server sends full chunks only when your client is sending player status update packets (any of Player (0x03) through Player Position And Look (0x06)).

## …all connecting clients spasm and jerk uncontrollably!

For newer clients, your server needs to send 49 chunks ahead of time, not just one. Send a 7×7 square of chunks, centered on the connecting client's position, before spawning them.

## …the client is trying to send an invalid packet that begins with 0xFE01

The client is attempting a legacy ping, this happens if your server did not respond to the Server List Ping properly, including if it sent malformed JSON.

## …the client disconnects after some time with a "Timed out" error

The server is expected to send a Keep Alive packet every 1-15 seconds (if the server does not send a keep alive within 20 seconds of state change to Play, the client will disconnect from the server for Timed Out), and the client should respond with the serverbound version of that packet. If either party does not receive keep alives for some period of time, they will disconnect.

## …my client isn't sending a Login Start packet!

By default, the Notchian server and client may group packets depending on if Nagle's TCP algorithm is enabled - the primary objective of Nagle's algorithm is to reduce the total number of packets needed to be sent over the network, increasing the efficiency. Nagle's algorithm is achieved by delaying each TCP packet to check if there are any more that are about to be sent to group them. You may not see a Login Start packet as you may not have parsed anything past the packet's length, because of this, you'll need to separate packets based off of the packet length.

# How do I open/save a command block?

The process to actually open the command block window clientside is somewhat complex; the client actually uses the Update Block Entity (0x09) packet to open it.

First, the client must have at least an OP permission level of 2, or else the client will refuse to open the command block. (The op permission level is set with the Entity Status packet)

To actually open the command block:

1. C→S: Player Block Placement (0x1C), with the position being the command block that was right-clicked.
2. S→C: Update Block Entity (0x09), with the NBT of the command block.

And to save it, use the MC|AutoCmd plugin channel. 
