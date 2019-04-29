# EVandroidLib
  
Library that enables quick start/access for communicating with Enterprise Voice skills.  

  
## Getting Started 

These instructions will help you implement this package in few easy steps. 


### Prerequisites 

Import package by following next steps in your Android Studio project:
- Click on File -> New -> New Module... -> select import .JAR/.ARR Package and specify the path to package
- in your app build.gradle in dependencies add:

``` 
dependencies {
    ...
    implementation project(":evandroidlib-debug")
    ...
}
``` 

- in settings.gradle file, change:

``` 
include ':app'
``` 

to: 

``` 
include ':app', ':evandroidlib-debug'
``` 
 
 sync your project, and it should complete successfully.
  

### Init 

First step is to create an instance of EVClientLoginData and populate it with keys you were provided: 

``` 
  val data = EVClientLoginData(
          EVRegion.US_EAST_1,
          identity_pool_id,
          user_pool_id,
          "bot_name",
          "bot_alias",
          "application_id"
  )
``` 

All responses are passed through EVInteractionDelegate so we should implement it at this point: 

``` 
... 
class TestActivity: AppCompatActivity(), EVInteractionDelegate {
... 
``` 

Interface implements two simple methods: onResponse(response: EVInteractorResponseModel) and onError(error: String). 
As the names suggest onError provides string response when an error occurs and onResponse method provides bot response data on successful interaction but more about response format a bit later. We should first finish with initialization. 

Main interaction interface is EVInteractionKit which we initialize by passing created EVClientLoginData object together with user identification; user tokens OR by directly logging in the user. In this step we also need to set our Activity that comforts to EVInteractionDelegate interface as a delegate. 

Example of passing tokens: 
 
``` 
val interactionKit = EVInteractionKit(
                     context: this,
                     clientLoginData: data,
                     accessToken: "",
                     idToken: "",
                     interactionDelegate: this) 
``` 
  
Example of username/password: 
  
``` 
val interactionKit = EVInteractionKit(
                     context: this,
                     clientLoginData: data,
                     username: "",
                     password: "",
                     cognitoAppId: "",
                     interactionDelegate: this)
``` 

If any of values in data object are wrong or invalid, you should get an error in onError delegate method. 


## Usage 

Now that our interaction kit is ready let's make a simple text request to our bot: 

``` 
interactionKit.textRequest("Help", mapOf("testAttr" to "testVal"))
``` 

In our request we pass a simple message and if we need to, we can add new key, value pair to session attributes. 

If all goes well, you should get a response in your onResponse method. Response data object is EVInteractorResponseModel with following keys: 

- responseType: it can be text or voice, depending on request type 
- sessionAttributes: key value pairs of attributes used in preserving interaction session; custom values form your bot can be found here 
- outputText: textual response to our request 
- inputTranscript: in the case of voice request, in this field you will find String representation of what bot understood 
- audioStream: in the case of voice request, Data object containing voice response; it can be played using this snippet: 

``` 
fun playVoiceResponse(response: InputStream) {

    player = MediaPlayer()

    player.setOnErrorListener(object: MediaPlayer.OnErrorListener {
        override fun onError(mp: MediaPlayer, what: Int, extra: Int): Boolean {
            val error = String.format(Locale.US,
                    "MediaPlayer error: \"what\": %d, \"extra\":%d",
                    what,
                    extra)
            System.out.println(error)
            return false
        }
    })

    player.setOnPreparedListener {
        player.start()
    }

    player.setOnCompletionListener {
        player.stop()
        player.release()
    }

    val tempAudioFile = File.createTempFile("lex_temp_response", "dat", context.filesDir)
    tempAudioFile.deleteOnExit()

    val audioOut = FileOutputStream(tempAudioFile, true)

    val buffer = ByteArray(512)
    var len = 0
    while (len != -1) {
        len = response.read(buffer)
        if ( len != -1) {
            audioOut.write(buffer, 0, len)
        }
    }
    audioOut.close()
    val audioIn = FileInputStream(tempAudioFile)
    player.setDataSource(audioIn.fd)
    player.prepare()
}
``` 

Few more notes regarding playback: 
  - player should be defined outside methods to avoid it being deallocated when the method finishes 
  - MediaPlayer doesn't support direct play of input streams so it needs to be saved to temp file and played from that file
 
