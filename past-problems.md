# Past Problems

The problems described in this page have been implemented and merged into the main project long enough to be considered settled.

## Spawning/Debugging

The original debug loop for API-driven resources like Portals was this:

1. Manually insert/update record. I was either using Swagger to PUT a record in the web browser or manually making SQL calls (even less ideal because that's what the API is supposed to be for).
2. Restart map server.
3. Log back in on client.
4. Navigate back to in-game location if necessary.
5. Observe results.
6. Repeat.

This was not an ideal loop. On top of being a slog in general, a few other points:
* If I'm using the built client I'm currently blind about where I should place the resource.
* A goal of this project is to make a workflow that would support multiple developers. Using the above loop would be mean that only one developer could be debugging an entire map of unknown size at a time.

This ended up being be an ideal use of the in-Unity HTTP server. New workflow:

1. Define a series of debug endpoints, such as:
    * List active maps/instances
    * List players
    * Upsert Portal
2. The existing `PortalSpawner`/`SpawnPortalGameEvent` combo became something like `PortalSetter`/`SetPortalGameEvent`:
    * The server maintains a registry of loaded portal entities on a map
    * If specified in the event, then also send a new `SavePortalGameEvent` that executes a PUT the portal record into the database
3. Debug endpoints can be enabled/disabled via configuration toggle. They're intended for development testing and not for use in a production environment.

Making endpoints means that I can make scripts to edit the game live, reducing my debug workflow to "Insert/Adjust -> Observe -> Repeat".

All of this can be used as a model modify other similar resources live.