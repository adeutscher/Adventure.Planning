# "Adventure" Game Framework Research Project Planning

This repository is an ongoing scratchpad of items that I'm currently grappling with in my game design research project.

For items that are more complete, see the [Adventure.Report](https://github.com/adeutscher/Adventure.Report) repository.

## Spawning/Debugging

At present, the debug loop for API-driven resources like Portals is this:

1. Manually insert/update record. I've been either using Swagger to PUT a record in the web browser or manually making SQL calls.
2. Restart map server.
3. Log back in on client.
4. Navigate back to in-game location (at least until player saving is implemented).
5. Repeat.

This is not an ideal loop. On top of being a slog in general, a few other points:
* If I'm using the built client I'm currently blind about where I should place the resource.
* A goal of this project is to make a workflow that would support multiple developers. Using the above loop would be mean that only one developer could be debugging an entire map of unknown size at a time.

This would be an ideal use of the in-Unity HTTP server.

1. Define a series of debug endpoints, such as:
    * List active maps/instances
    * List players
    * Upsert Portal
2. The existing `PortalSpawner`/`SpawnPortalGameEvent` combo would become something like `PortalSetter`/`SetPortalGameEvent`:
    * Maintain a registry of loaded portal entities (I already have something called the `EntityRegistry` but I'm not fully leveraging it yet)
    * If specified in the event, then also send a new `SavePortalGameEvent` that would PUT the portal record into the database
3. Debug endpoints would be enabled/disabled via configuration toggle. They're intended for development testing and not for use in a production environment.
