#Full Stack NanoDegree Project 4
##Conference Central Sessionized
### author Andrew Roy Chen
### last modified 2015 sept 14

### tech and services
- [App Engine][1]
- [Python][2]
- [Google Cloud Endpoints][3]

## project notes
###source
built on the Udacity Conference App found [in this repo](https://github.com/udacity/ud858/tree/master/ConferenceCentral_Complete).

###setup and run

1 Update the value of application in app.yaml to the app ID you have registered in the App Engine admin console and would like to use to host your instance of this sample.

2 Update the values at the top of settings.py to reflect the respective client IDs you have registered in the Developer Console.

3 Update the value of CLIENT_ID in static/js/app.js to the Web client ID

4 (Optional) Mark the configuration files as unchanged as follows: $ git update-index --assume-unchanged  app.yaml settings.py static/js/app.js

5 Run the app with either:
 - the devserver using dev_appserver.py DIR or,
 - using the App Engine client
and ensure it's running by visiting your local server's address (by default localhost:8080.)

6 (Optional) Generate your client library(ies) with the endpoints tool.

7 Use endpoints not reachable in the Front-End UI by visiting the project's API explorer
page, default found at
```
localhost:8080/_ah/api/explorer
```

8 Deploy your application.


###task 1
Session implementation and justification of implementation decisions for the
chosen data types:

I followed project guidelines to the letter in defining the Sessions model.
For the Message form object I added a property *websafeConferenceKey* to
easily access specific Conference entities.

Most of the properties were no-brainer String types and this includes *name*,
*highlights*, *typeOfSession*. The latter is interesting in that the front-end
should ideally incorporate a list of accepted values, the same way T-Shirt sizes
are specified.

*Date* is, unsurprisingly, a DateProperty translating to a Python datetime
object.

*StartTime* likewise is implemented as a TimeProperty but this decision has
interesting ramifications on filtering by this property. In short, perhaps an
Integer type would be more functional. This is discussed in section **Task 3**.

*Duration* is expressed as an Integer because realistically they will represent
a specific minute count. This makes comparisons and filtering straightforward.

Finally, we have *speaker*. I picture User access of Session information in the
client from a Conference *Details* page.

To represent the session *speaker* I just went with a String. I thought about
making a specific Entity. Problem is from a User/Session creator point of view,
introduces the burden of defining additional information on the speaker that the
conference creator may not know.

Also it is possible that different conference creators may end up creating
multiple speaker entities for the same person, perhaps with slightly different
spellings or other minor differences that ultimately just clog the store.

Separate 'speaker' entities make more sense if there is a UI implementation that
shows existing 'speaker' profiles. Perhaps a UX flow would allow a session
creator to start typing a name and be presented with already created 'speakers'
of similar spelling. Regardless, though implementation of a Speaker entity would
be trivial, it did not feel like the best choice given the client.


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

One method to work around the restriction is to combine a property filter for
session type with some good ol' Python:
```
good_sessions = Session.query(Session.typeOfSession != 'Workshop').fetch()
if good_sessions:
    cutoff = time(19)
    good_sessions = [session for session in good_sessions if session.startTime < cutoff]```
```
The code above will manually filter the non-'Workshop' type sessions with a
handy list comprehension by comparing datetime.time objects. My concern is perf
with suitably large data sets.

This solution is implemented as the endpoint *task3Test*.

**Another possible solution** is to save the *startTime* property as an integer val
(asking the user to enter the time on a 24-hr scale, ,then converting). This
would make filtering possible. As it is, my efforts to filter by TimeProperty:
```
good_sessions.filter(Session.startTime < cutoff)
```
did not work correctly. Although datatstore did not complain, the results failed
to filter correctly. Perhaps datetime objects can not be adequately compared in
queries?


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
