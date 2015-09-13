#Full Stack NonoDegree Project 4
##Conference Central Sessionized
### author Andrew Roy Chen
### last modified 2015 sept 12

### tech and services
- [App Engine][1]
- [Python][2]
- [Google Cloud Endpoints][3]

## project notes
###task 1
Session implementation:

I picture User access of Session information in the client from a Conference
*Details* page.
I followed the project guidelines to the letter in defining the Sessions model.
For the Message form object I added a property *websafeConferenceKey* to
easily access specific Conference entities.


Speaker implementation:

To represent the session *speaker* I just went with a String. I thought about
making a specific Entity. Problem is from a User/Session creator point of view,
introduces the burden of defining additional information on the speaker that the
conference creator may not now.

Also it is possible that different conference creators may end up creating
multiple speaker entities for the same person, perhaps with slightly different
spellings or other minor differences that ultimately just clog the store.

Separate 'speaker' entities make sense if there is a UI implementation that
shows existing 'speaker' profiles. Perhaps the UX flow would allow a session
creator to start typing a name and be presented with already created 'speakers'
of similar spelling.


###task 2
Session Wishlists:

I modified the Profile model to have a list of Session keys, following the
example of keeping a list of Conference keys. The way I envisioned UX flow, from
User profile page there could be a button or link to view all wishlisted
sessions. So retrieving the list would be a simple matter of fetching Session
entities given the currently logged-in User's Profile data.

One key decision was whether to make Profile updates transactional. I finally
decided to leave off transactions because adding a session to wishlist is
a personal action unique to a User. Updating attendence needed to take into
consideration that multiple Users could be accessing the db at the same time.
This is not, or should not, be the case with wishlist modifications.


###task 3
Additional Queries:

-*Can get conferences with available seats*
I picture the User being able to just check which conferences even have space
to register. Much better than hitting 'register' only to be disappointed.

Implementation as it's own endpoint is probably overkill. Better for the front-
end to just configure a call to *queryConferences*. That said I implemented it
under the endpoint *getConferencesWithSeats*. Simply runs a property query for
*availableSeats* > 0.


-*Can get sessions less than 1 hr*
It would be nice to just browse a list of shorter sessions, something to drop
in on that won't demand a significant time investment.

Since there is no handy-dandy *querySessions* endpoint, I implemented this as
*getShortSessions*. It simply returns a property query on Sessions with duration
 less than 60 min.


A query-related problem:

What if User wanted to avoid workshop (type) sessions as well as sessions after
7pm? What are the issues here and how could this be implemented?

The first problem is that datastore allows a max of one inequality filter per
query. So in the above case, two inquality filters are needed:
   - typeOfSession != "workshop" &&
   - startTime < 7
Where something like:
```
   Session.query(typeOfSession != "workshop" && startTime < 7)
```

is *not allowed*.

One method to work around the restriction is to chain filter() calls.
```
   Session.query().filter(typeOfSession != "workshop").filter(startTime < 7)
```


###task 4
Implement getFeaturedSpeaker endpoint:

I modified createSession to check every time a new session is created. If a
'featured speaker' condition was found, I pushed a task to the default taskqueue
along with parameter data for websafe conference key and the speaker's name.

The taskqueue in turn triggers a handler defined in main.app which finally
calls ConferenceAPI's *_cacheFeatured* helper method, passing in the params.

Within the helper method, the speaker and session names are extracted and put in
a string to be saved to Memcache.

A User would then need only to call the endpoint *getFeaturedSpeaker*, which
takes no parameters, to retrieve the data from Memcache.

Two notes from this task:

- The implementation seems roundabout but apparently we cannot directly call
endpoint methods from the task push queue as detailed here:

[Taskqueue docs] (https://cloud.google.com/appengine/docs/python/taskqueue/overview-push?hl=en#Python_calling_google_cloud_endpoints)

- Tasks default to POST requests...a gotcha that had me blocked for a few days
as I tried to figure out why my GET handlers defined in *main.py* were never
executing.
