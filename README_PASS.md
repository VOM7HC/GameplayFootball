**GameplayFootball Extension for PASS**

This is an extension of GameplayFootball for PASS, it generates a JSON file with game data for each game played.
This game data can then be used to create a report with PASS.

**StoreMatchAction**

The function StoreMatchAction is called whenever a special action has happened, e.g. kickoff, goal kick, penalty.
They are then stored inside a list as a ptree item.

**MatchLineup**

The function MatchLineup is called when the match has ended, it goes over both teams, and for each team it creates the lineup with all the players.

**MatchInfo**

The function MatchInfo is also called when the match has ended, it creates the MatchInfo field inside the JSON.

**MatchAction**

The function MatchAction takes all the stored ptrees by StoreMatchAction and puts them inside another ptree.

When the match has ended the functions: **MatchLineup, MatchInfo, and MatchAction** are called, each function returns a ptree.
These ptrees are then added as a child to a parent ptree, lastly, this ptree is written to a JSON file.

Note that the date field normally looks like: "/Date(1643410787730)/".
However, the **write_json()** method replaces the forward slash by a backward slash followed with a forward slash,
which causes the date field to look like: "\/Date(1643410787730)\/"