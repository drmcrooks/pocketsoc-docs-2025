# MISP

Having generated some logs using zeek, we're now going to explore MISP.

1. Use the link and credentials on the front page to access your MISP website
![MISP login](images/misp_1.png)
2. You should see an "Events" page that's blank
![MISP events](images/misp_2.png)
3. If you click on *Automation* at the bottom of the list of options on the left-hand side, you can find a very detailed set of docs on the API access for MISP.
5. Before we create an event, we want to make sure that all our events will have a TLP setting. Go to "Event Actions" and select "Taxonomies"
![MISP taxonomies](images/misp_4.png)
6. We want to require TLP and also highlight it: this will make it easier to select when creating an event. Click these two options for TLP and then the "Play" icon to enable (mousing over will confirm which icon to use): click OK to confirm. Then click "enable all" to enable all the tlp options.
![MISP enable](images/misp_4a.png)
7. Repeat for PAP
7. Let's now explore the "add event" option: this will allow us to generate an event based on the findings of our "malicious" webserver
![MISP add](images/misp_5.png)
8. There are a number of options here: let's look at them:
    - Date: Set the date to when the event is being recorded or occurred
	- Distribution:
	    - Your organisation only: will not be shared outside this instance
		- This community only: will only be shared with instances in your community
		- **Connected communities: will be shared with any communities connected to yours but no further**
		- All communities: will be shared with any MISP instance any number of hops from you
	- Threat level: severity of the event
	- Analysis: allows you to specify whether these are your initial findings, part way through an extended analysis, or form the completed version of your findings
	- Event Info: A brief description of the event
	- Extends Event: Not used for a new incident, but could be used if you are basing your findings on another event
9. For Distribution, this can typically be used to set how far your event will be propagated
10. Enter some explanatory text and options, and click "Submit". You will see some metadata about this event, with some useful warnings about things you may want to add to your event, like attributes!
![MISP metadata](images/misp_6.png)
11. We're going to start by adding a TLP tag to our event to indicate the sharing level: this is good practice so that it's always clear which is why we have required it for all of our events. 
![MISP tags](images/misp_7.png)
12. We're going to use `tlp:amber`: choose from the dropdown and Submit.
13. Repeat for PAP:AMBER
14. Good practice is, rather than adding individual attributes, to add *objects*. These allow you to add a set of related attributes - such as all the information about a file, for example, or all information about a URL or domain. 
15. We'll start by adding a "network" object. Go to Add Object, and start typing "network". Click on that option and you'll see another option dropdown - start typing "domain" and choose "domain:ip"
![MISP domainip](images/misp_8.png)
16. This page lets us fill in all the information we have about given domain/ip. We can add:
	- IP: (the IP you identified earlier)
	- You might also add first seen/last seen as today
17. It's good practice to put comments wherever you have the option to, so add comments both to the object itself and the individual attributes, then click Submit. You'll see a review of the information, then click "Create new object" if you're happy. MISP will attempt to validate your entries; if it finds an issue it will alert you before you can proceed.
![MISP domainip](images/misp_9.png)
18. You can now see the attributes in the main event page. Note that MISP has highlighted the date in red as these attributes have not been "published" yet; i.e. it will not be propagated to sharing groups, and the attributes will not be available via the API
![MISP domainip](images/misp_10.png)
19. MISP uses the idea of working on an "unpublished" event until it has been completed to a certain level: the full results of an initial analysis, for example, or attributes that have not been reviewed by a user with the rights to publish. This means that a team can work on an event safe in the knowledge that the attributes will not be published until a team leader, for example, has provided a cross check.
20. To publish, click "Publish no email" on the left hand side. While we would normally inform users of a new event, we have no other users.
21. If you return to the home screen and select the event again, you'll see that the dates attached to the attributes are no longer highlighted since the event has been published.
22. You can also add additional objects - for example look for the "file" objects where you can add checksums and other file details.
23. Click Publish (no email) - you can review the impact of this change on any remote servers to which you are attached





