---
layout: post
title:  "Using the Amazon Echo to Control VLC and YouTube"
date:   2015-07-18 23:02:14
categories: Amazon Echo, VLC, Home Automation
---

The [Amazon Echo][amazon-echo] is a fantastic step in voice-controlled assistants. I've been calling it my "assistant cylinder". In a normal home environment there wasn't very much noise and Alexa recognizes almost every command given to her. I spent some time setting up my [Phillips Hue][smart-lights] and enjoying that I could now control my lights with my voice, something that I felt was straight out of a science fiction story. 

Having an Amazon Prime account, I did get Alexa to play some music from the Amazon Prime Music catalog, but I felt that it could be better. For one, I'm using the [Echo][amazon-echo] in my living room, where I've already got my television media center and speakers wired up and music from them sounds great. Secondly, the selection in the free part of the Prime Music catalog could be better. I asked it for a decent amount of songs that it could only play samples of. These were really my only complaints about the Echo. I wanted to use it to control my own speakers, and to select the content source. As a programmer, I felt that thanks to the [Alexa Skills Kit][alexa-skills], I could accomplish this myself. And I have done just that. I can now tell Alexa to tell my media center to play anything from my media library or anything from YouTube and have it play on my television instantly through [VLC][vlc]. It has support for next, previous, stop, resume, and other commands that can control your media. No more keyboard or mouse required!

### But How?

How did I accomplish this witchcraft? Thanks to the efforts of projects such as [VLC][vlc], [OpenCV][opencv], [fuzzywuzzy][fuzzy], and the [YouTube API][youtube-api], as well as [Amazon's Lambda][lambda], we were able to make everything talk to each other and combine to make a great thing. 

##### Disclaimer:


This code worked for me on my system. It may not work for you on your system. If you don't understand what the code does, do not use it.

Looking for the code right away? [Find it here, on GitHub!](https://github.com/zioyero/mediaCenterSkill)

#### [Alexa Skill][alexa-skills]

Amazon's Lambda services provide a great and easy way to set up an [Alexa Skill][alexa-skills]. The skill takes a basic structure of "Alexa, tell **kickass** to **play Elements by Lindsey Stirling**", where **kickass** is the *Invocation Name* of the skill, which tells Alexa that she needs to pass off the command to a particular skill and everything after the *to* is the command that gets sent to the skill via an Intent. In this case, it would send a command to our media server to do a YouTube search for "Elements by Lindsey Stirling" and to play the first result. 

Intent? Yes, each skill can handle different types of commands. For example, in our media center skills kit, we have seperate commands for **next** and **previous**, each of which is defined by a particular Intent. For the simple example of the next intent, it has a schema that looks like:

{% highlight JSON %}
{
  "intents": [
    {
      "intent" : "NextIntent",
      "slots": []
    },
    {
      "intent" : "PreviousIntent",
      "slots": []
    }
  ]
}

{% endhighlight %}

And they were bound to the utterance patterns of:

{% highlight text %}
NextIntent next
NextIntent go to the next song
NextIntent skip this song
NextIntent skip
PreviousIntent go back
{% endhighlight %}

What this means is that whenever Alexa heard "Alexa, tell the media center to skip", it would pick out **skip** as the command, and understand that it means it should trigger the `NextIntent` intent. 

Now this is where [Amazon Lambda][lambda] comes in. The skill can be configured to either call an endpoint on your own server when the skill is triggered by Alexa with the command spoken, or it can execute a lambda function, hosted on Amazon's own servers. For our purposes, we didn't need any fancy server so we chose to use the lambda function. It was surprisingly easy to set up. 

#### [Amazon Lambda][lambda]

Setting up a lambda function was very easy, and they provice an in-browser code editor and tester that proved useful. The logs for the function are also stored in CloudWatch, which is very convenient. When the lambda function is triggered by Alexa, it sends a JSON request with the intent that was triggered, and any paramaters that were part of that intent. An example for triggering the `NextIntent` would look like:

{% highlight JSON %}
{
  "session": {
    "new": false,
    "sessionId": "session1234",
    "attributes": {},
    "user": {
      "userId": null
    },
    "application": {
      "applicationId": "amzn1.echo-sdk-ams.app.[unique-value-here]"
    }
  },
  "version": "1.0",
  "request": {
    "intent": {
      "slots": {},
      "name": "NextIntent"
    },
    "type": "IntentRequest",
    "requestId": "request5678"
  }
}
{% endhighlight %}

And since the lambda function is written in Node.js, it was very easy to parse out the intent name that we needed and trigger the proper intent. From there, it was just a matter of handling the command and telling Alexa what to say as a recognition of the command.

Handing the command was very easy after parsing out the intent.

{% highlight javascript %}
/**
 * Called when the user specifies an intent for this skill.
 */
function onIntent(intentRequest, session, callback) {
    console.log("onIntent requestId=" + intentRequest.requestId
                + ", sessionId=" + session.sessionId);

    var intent = intentRequest.intent,
        intentName = intentRequest.intent.name;

    // Dispatch to your skill's intent handlers
    if ("RelayIntent" === intentName) {
        relayMedia(intent, session, callback)
    } else if ("NextIntent" === intentName){
        nextIntent(intent, session, callback);
    } else if ("PreviousIntent" === intentName){
        prevIntent(intent, session, callback);
    } 
    else {
        throw "Invalid intent";
    }
}

function nextIntent(intent, session, callback){
    console.log("Relaying next command for intent: ");
    console.log(intent)
    var query = {
        next: true
    }
    var speechResponse = "Next";
    sendToMediaCenter(query, speechResponse, callback);
}

/**
 * Sends a query to the media center and instructs alexa to utter the given speech output.
 */
function sendToMediaCenter(query, speechOutput, callback){
    var httpQuery = queryString.stringify(query)
    var httpCallback = function(response){
        console.log('done');
        callback({}, buildSpeechletResponse("Relaying", speechOutput, null, true))
    }
    console.log("Making request to")
    var options = {
        host: "yourComputerIP",
        port: yourPortNumber,
        path: "?"+httpQuery
    }
    console.log(options)
    http.request(options, httpCallback).end();
}

{% endhighlight %}

All that code does is create an http request with a single query parameter, `next` and makes the request to the server running on our media center. It then sends a response back to Alexa to tell her to speak the word "next", and that this ended the interaction with the skill. It then becomes the task of the media center to respond to this command by instructing vlc to go to the next item in the playlist.

Running a simple http server using `BaseHTTPServer`, we set up a server that only handles `GET` requests on a single port and delegates out the commands according to the query parameters. Basic server stuff, really. Continuing with the example of the `next` command, the server listens forever on a port and keeps an instance of [VLC][vlc] alive to use it as the front-end media player. 

{% highlight python %}
vlc = Popen(["/Applications/VLC.app/Contents/MacOS/VLC", "-I", "macosx", "--extraintf", "rc"], stdin=PIPE)

class MyHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        parsed = urlparse.urlparse(self.path)
        self.send_response(200)
        self.send_header("Content-Type", "text/html")
        self.end_headers()
        self.wfile.write("done")
        args = urlparse.parse_qs(parsed.query)
        print(self.path)

        handleCommand(args)


def handleCommand(args):
    if "youtube" in args:
        playFromYoutube(args["youtube"][0])
    elif "next" in args:
        vlc.stdin.write("next\n")
    elif "prev" in args:
        vlc.stdin.write("prev\n")
{% endhighlight %}

And that's it! Really. That's all that's required to get Alexa to communicate with your media center computer. This does, of course, assume that your media center is a computer running with [VLC][vlc] installed. 

#### Where's the Content?!

I know, talking to VLC is cool and all, but I've only shown you how to get Alexa to get VLC to skip around a playlist. How about adding some content? This is really what we wanted, too. First, let's go over how to get content from [YouTube][youtube] (which, let's face it, is the greatest catalog of audio/visual content that exists).

First, we had to get Alexa to give us a wildcard parameter, allowing for an interaction like "Alexa, tell **kickass** to play **YouTube Video**". This wasn't as easy as I expected, but it was definitely possible. I set up a new Intent to relay the query to the media center, the `RelayIntent` . 

{% highlight JSON %}
{
  "intents": [
    {
      "intent" : "RelayIntent",
      "slots": [{
      	"name" : "query",
      	"type" : "LITERAL"
      }]
    }
  ]
}
{% endhighlight %}

I also set up some an utterence to trigger this skill:

{% highlight text %}
RelayIntent play {video | query}
{% endhighlight %}

The syntax of `{video | query}` defines the `query` parameter with a default or sample value and tells Alexa where in the utterence to look for the parameters. Sadly, I realized soon after this that the number of words defined in the utterence samples does matter. I was only getting one word in my search query, usually the last word. So it had to be changed to take in any number of words. At least, it should have been any number of words. I could not find a way to get Alexa to greedily capture everything after a word as a parameter. Instead, I applied the following hack:

{% highlight text %}
RelayIntent play {title | song}
RelayIntent play {title title | song}
RelayIntent play {title title title | song}
RelayIntent play {title title title title | song}
RelayIntent play {title title title title title | song}
RelayIntent play {title title title title title title | song}
RelayIntent play {title title title title title title title | song}
RelayIntent play {title title title title title title title title | song}
RelayIntent play {title title title title title title title title title | song}
RelayIntent play {title title title title title title title title title title | song}
RelayIntent play {title title title title title title title title title title title | song}
RelayIntent play {title title title title title title title title title title title title | song}
RelayIntent play {title title title title title title title title title title title title title | song}
{% endhighlight %}

And it worked surprisingly well. I haven't ran into an issue with queries being cut off anymore. So now that I was able to capture whole queries, I needed to pass them to the media center again. In the same file as the javascript code above, I added the `relayMedia` function to do just that. 

{% highlight javascript %}
function relayMedia(intent, session, callFunc){
    console.log("Relaying media for intent: ")
    console.log(intent)
    var songName = intent.slots.song.value
    var query = {
        youtube: songName
    };
    var speechResponse = "Sending "+songName+" to media center";
    sendToMediaCenter(query, speechResponse, callFunc)
}
{% endhighlight %}

Which simply calls the same endpoint as before with a different query parameter, and the spoken query as the value, allowing the media center server to get the full query string, having been translated from speech to text by Alexa. Now the media center had to do a YouTube search with the query, resolve the results into a URL and then pass it to VLC. Thankfully, there were libraries available to access the [YouTube API][youtube-api] and the VLC command-line interface is already quite fantastic, so this became an easy task. Using the python YouTube library, we developed the following functions:

{% highlight python %}
youtube = build("youtube", "v3", developerKey = "getYourOwn")
def playFromYoutube(query, queryType = "video"):
    print(query, queryType)

    response = youtube.search().list(q=urllib.unquote(query), part="id,snippet", maxResults=5, type=queryType).execute()

    results = response.get("items", [])

    if queryType == "video" and not len(results) == 0:
        playYoutubeVideos([results[0]["id"]["videoId"]])


def playYoutubeVideos(videoIds):
    vlc.stdin.write("clear\n")

    if not len(videoIds) == 0:
        videoUrl = "http://youtube.com/watch?v=%s" % videoIds[0]
        vlc.stdin.write("add %s \n" % videoUrl)

    for videoId in videoIds[1:]:
        videoUrl = "http://youtube.com/watch?v=%s" % videoId
        vlc.stdin.write("enqueue %s \n" % videoUrl)

{% endhighlight %}

Thanks to VLC already handling YouTube URLs, this took a lot of the work off of us.

#### Conclusion 

And there you have it! YouTube content at your disposal using only your voice. The YouTube search algorithm combined with Alexa's speech to text technology makes this interface a fantastic way to use your media center, and since setting it all up, we've spent many nights simply enjoying music videos and everything else YouTube has to offer in an interestingly social way. But more on that later. 


[amazon-echo]: http://smile.amazon.com/Amazon-SK705DI-Echo/dp/B00X4WHP5E/ref=sr_1_1?ie=UTF8&qid=1437287015&sr=8-1&keywords=echo&pebp=1437287018130&perid=1XFWKH7X88J9PHY3F1G5
[smart-lights]: http://www2.meethue.com/en-us/
[alexa-skills]: https://developer.amazon.com/public/community/post/Tx205N9U1UD338H/Introducing-the-Alexa-Skills-Kit-Enabling-Developers-to-Create-Entirely-New-Voic
[vlc]: https://www.videolan.org/vlc/index.html
[lambda]: http://aws.amazon.com/lambda/
[opencv]: http://opencv.org/
[fuzzy]: https://github.com/seatgeek/fuzzywuzzy
[youtube-api]: https://developers.google.com/youtube/
[youtube]: http://youtube.com


