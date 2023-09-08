# Intro to web and service workers
https://www.youtube.com/watch?v=OgLemdR65pE&pp=ygUfZGVteXN0aWZ5aW5nIHdlYiBzZXJ2aWNlIHdvcmtlcg%3D%3D

# Notes from above talk
- rendering > 60fps means no jankiness (=> code should not block the dom for more than 1000ms/60fps = **16.7ms**)
- web workers (parallelism). For ex: writing to indexedDb, ajax calls, user interaction, response.json() etc.
- service workers make websites work offline (push notifications, serve data from cache first and then network)


