# Discussion

The basic assumption is that the two players must be able to play remotely, so that they don't have to care about hiding their choices before each other.
The remote setup implies the following minimal requirements:
* interaction through HTTP requests
  * requests must be blocking, both in the sign-up scenario (waiting for the second player to join) and in the turn scenario (waiting for the second player's choice)
  * since the sign-up scenario is not specified in detail, following options must be supported:
    * player-computer
    * two players knowing the names of each other
    * two players ready to play with anyone available
* persistent remote storage of the game progress
  * a DB with the list of games and either their current scores or their entire turn histories
  * is used for the synchronization of the game sign-up and game turns
and additionally on top of the basic functionality:
* user authentication to prevent turn requests on behalf of the opponent
* listing of available players during the sign-up
* periodic purging of old games

The option to play with a computer doesn't bring up any additional complexity into the scenarios (on contrary, such a use case could be implemented locally, without any remote infrastructure).

# Scope

Given the suggested time limit and complexity of the problem, I'm not able to provide even a draft of the solution.
The only feasible output is a list of tasks I'd expect to work through in order to deliver an MVP.

# Backlog Items

## Necessary
1. Project setup

   *What*: Configuration of packaging, linting and testing tools

   *Why*: To ensure the minimal support for the development and deployment, project must be deliverable and the code must quality must be ensured in a repeatable way

   *How*: Provide `pyproject.toml` configuration for `poetry`, using `poetry` for dependencies and virtual env management, `setuptools` for packaging, `black` for auto-formatting, `pylint` for linting, `mypy` for type-checking and `pytest` as a test harness.
   Generate initial `poetry.lock` file.

2. Infrastructure setup

   *What*: Dockerfile for the service image 

   *Why*: To enable instalation of the service on a local machine/cloud infrastrucuture

   *How*: Derive an image from e.g. `python-3.10:slim` that copies the relevant repo content, installs the package and starts the service and the DB.

3. Data model

   *What*: Representation of the state of games

   *Why*: To enable the evaluation of the game turns and saving of the game scores

   *How*: Create an `sqlalchemy` mapping for:
   * `game`: uniquely determined by the pair of alphabetically sorted usernames and a timestamp, contains a persistence flag `saved` (whether the game has been saved and thus must not be purged during a periodic clean-up).
   * `game-state`: with a game-id, last choice (rock/paper/scissors) of both players, current score of both players

4. Sign-up scenario

   *What*: Enable a user to sign-up for a game with a computer, a particular player or any player

   *Why*: Necessary for the basic functionality

   *How*: blocking `GET` request:
   * user-vs-computer: `endpoint/signup/name1`
     * create a `game` record with `computer-name1-TIMESTAMP` id and initialize the corresponding `game-state` 
     * return the game id
   * user-vs-specific-user: `endpoint/signup/name1?with=name2`
     * if an *unsaved* `game` with the two usernames in the alphabetical order does not exist yet, create it with the current timestamp
       * block until a record for this game id appears in `game-state`
     * otherwise initialize the corresponding `game-state`
     * return the game id
   * user-vs-any-user: `endpoint/signup/name1?with=any`
     * if a `game` with a single username (for the first player) filled in does not exist yet, create it with the current timestamp
       * block until a record for this game id appears in `game-state`
     * otherwise update the record the existing game record with `name1` as the second player and initialize the corresponding `game-state`
     * return the game id

5. Turn scenario

   *What*: Enable user to make a turn and report the result back

   *Why*: The basic step of the game

   *How*: blocking `GET` request `endpoint/play/GAME-ID/name1/[rock|paper/scissors]`
   * update the `game-state` record with the choice of player `name1`
   * block until the choice of the other player appears in the record
   * increment the score of the winner of the tur
   * return the username of the winner and the updates scores of the both players

6. Save scenario

   *What*: Enable to end the game and persist the results

   *Why*: Supports retrieval of game history and allows to start a new game with the same opponent later on 

   *How*: `GET` request `endpoint/save/GAME-ID`
   * set the `saved` flag on the corresponding `game` record
   * return the game id 

## Nice-to-have
1. User authentication

   *What*: Add a token mechanism replacing the usernames in the requests

   *Why*: To prevent cheating by sending turn requests on behalf of the opponent

   *How*: Sign-up request returns a randomly generated token (`UUID`) associated with the user that will be send as a part of the valid `play` requests.

2. User-friendly sing-up

   *What*: A two-step sign-up process

   *Why*: Allow users to pick their opponents from a list

   *How*: In the preliminary sign-up step, the request registers the username and returns the list of all waiting players 

3. Purging of old games

   *What*: Cleanup of game records with the timestamp below threshold

   *Why*: Save storage space and thus infrastructure costs 

   *How*: A cleanup with some reasonable time delta