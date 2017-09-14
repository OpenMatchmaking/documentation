# Architecture

How it works
------------
The matchmaking services are working by the following schema:

<p align="center">
  <img src="https://github.com/OpenMatchmaking/pathfinder/blob/master/docs/images/Architecture.png"/>
</p>

As you can see at this image it contains three different components:
1. **Game client**. Represents a certain "unified shell" for each active user, which is connecting to the main game server and exchanges data with it if necessary.
2. **Game server**. Represents an existing node in a cluster, which transmits data about its internal state of the particular game and allow its connected clients to maintain their own accurate version of the game world for display to players. Also could processing/receiving each player's input, messages and so on.
3. **Open Matchmaking platform**. That is the core component of our platform, to what we are aiming for. It do everything that related to the matchmaking and that is expected from this service: 
    - Searching an opponent with similar skills (or a suitable team, if it is a competitive game that focus on the teamplay) for each player, so that the chances for will be close to the 50/50 percents on win/lose. Which is bringing a lot of challenges, opportunites to improve its skills and fun for everyone in each match.
    - Providing a lot different strategies (or matchmaking algorightms), that could be used for different game modes and genres.
    - Collecting some statics, that could be used for further data processing, like creating the leaderboards, plays of the day and etcetera after ending each match.
