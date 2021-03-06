# Voice Gateway Tools - Network

This folder contains tools you can use to debug voice gateway networking related issues. The focus here is on tools that can help you identify hybrid-cloud networking issues that could be causing voice gateway issues like socket timeouts and call latencies.

## Watson Assistant Curl Script: wa-curl.sh

This bash script will issue periodic requests to Watson Assistant (default period is every 2 seconds) and will print out cases where the response time exceeds a predefined threshold (default threshold is 2 seconds). This WA request does not include input text so a new conversation-id will be generated on every request. The curl command generates all of the following:

- **HTTP Response Headers** - needed to access transaction IDs
- **JSON Response from Watson Assistant** - which includes the conversation ID
- **Time Elements** - which provide details about where time was spent during the transaction

You will need to make the following two changes to the script before running it:
1. Be sure to set the username:password to match your WA service instance (note that apikey:your-apikey will also work for username password).
1. Be sure to modify the WA service URL to match the URL associated with your service instance.

Note that this script first dumps all the results to a file (results.txt) and then uses awk to parse the last line of the file to get the total transaction time.

Here is an example of the response that is returned from this script:

```
time: 02/28/2019 15:47:47
HTTP/2 200
x-backside-transport: OK OK
content-type: application/json; charset=utf-8
access-control-allow-origin: *
access-control-allow-methods: GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
access-control-allow-headers: Content-Type, Content-Length, Authorization, X-Watson-Authorization-Token, X-WDC-PL-OPT-OUT, X-Watson-UserInfo, X-Watson-Learning-Opt-Out, X-Watson-Metadata
access-control-max-age: 3600
content-security-policy: default-src 'none'
x-dns-prefetch-control: off
x-frame-options: SAMEORIGIN
strict-transport-security: max-age=31536000;
x-download-options: noopen
x-content-type-options: nosniff
x-xss-protection: 1; mode=block
x-global-transaction-id: 7ecac92c5c7848f316aa17ed
x-dp-watson-tran-id: gateway01-380245997
x-dp-transit-id: gateway01-380245997
content-length: 411
date: Thu, 28 Feb 2019 20:47:47 GMT

{"intents":[],"entities":[],"input":{"text":""},"output":{"text":["Hi, my name is Watson. What can I do for you today?"],"nodes_visited":["Introduction"],"log_messages":[]},"context":{"conversation_id":"e13622f3-0816-4656-ad0f-5c187fefc5a4","system":{"initialized":true,"dialog_stack":[{"dialog_node":"Introduction"}],"dialog_turn_counter":1,"dialog_request_counter":1,"_node_output_map":{"Introduction":[0]}}}}
lookup:        0.061419
connect:       0.121267
appconnect:    3.288581
pretransfer:   3.288919
redirect:      0.000000
starttransfer: 3.288951
total:         3.423352
```
First is the HTTP response headers. Second is the actual JSON response from WA and then all the time elements. Here are the definitations for each of the time elements:

- **time**: The time that the curl command was issued.
- **lookup**: The time, in seconds, it took from the start until the name resolving was completed.
- **connect**: The time, in seconds, it took from the start until the TCP connect to the remote host (or proxy) was completed.
- **appconnect**: The time, in seconds, it took from the start until the SSL/SSH/etc connect/handshake to the remote host was completed. (Added in 7.19.0)
- **pretransfer**: The time, in seconds, it took from the start until the file transfer was just about to begin. This includes all pre-transfer commands and negotiations that are specific to the particular protocol(s) involved.
- **redirect**: The time, in seconds, it took for all redirection steps include name lookup, connect, pretransfer and transfer before the final transaction was started. time_redirect shows the complete execution time for multiple redirections. (Added in 7.12.3)
- **starttransfer**: The time, in seconds, it took from the start until the first byte was just about to be transferred. This includes time_pretransfer and also the time the server needed to calculate the result.
- **total**: The total time, in seconds, that the full operation lasted. The time will be displayed with millisecond resolution.

To run this script you will need to do the following:

1. If you are running on a MacOS, you will need to install the GNU implementation of awk. Its needed for the accessing systime. Its really easy to do this using **homebrew** like this: ```brew install gawk``` (note that this step is not needed on other Linux systems)
1. You will need to modify the wa-curl.sh file with your Watson Assistant details including credentials and workspace ID. Look for the following tags to modify ```username```, ```password``` and ```workspace-id```
1. If you want to change either the periodicity of the script or the latency threshold it should be easy to see in the script how to do that.
1. Run the script like this ```./wa-curl.sh > output.txt ```

**Note that this script currently supports the v1 Watson Assistant API. You can pass an API-KEY to it by setting the username to ```apikey``` and password to the actual api key value.**

## Text-To-Speech Curl Command Example

You can use this curl command to test Watson TTS:

curl -X POST -u "username":"password" \
--header "Content-Type: application/json" \
--header "Accept: audio/wav" \
--data "{\"text\":\"Hello world\"}" \
--output hello_world.wav \
--show-error \
--write-out 'lookup:        %{time_namelookup}\nconnect:       %{time_connect}\nappconnect:    %{time_appconnect}\npretransfer:   %{time_pretransfer}\nredirect:      %{time_redirect}\nstarttransfer: %{time_starttransfer}\ntotal:         %{time_total}\n' \
"https://stream.watsonplatform.net/text-to-speech/api/v1/synthesize?voice=en-US_AllisonVoice"


## Speech-To-Text Curl Command Example


Use the following curl command to print timing information as well as test the connection to Watson Speech-To-Text:

```
curl -X POST -u "apikey:{apikey}" --header "Content-Type: audio/wav" --data-binary @resources/smoky-fires.wav --write-out "\ndns_lookup: %{time_namelookup}\nconnect: %{time_connect}\nappconnect: %{time_appconnect}\n pretransfer: %{time_pretransfer}\nredirect: %{time_redirect}\nstarttransfer: %{time_starttransfer}\ntotal: %{time_total}\n" \
"https://stream.watsonplatform.net/speech-to-text/api/v1/recognize?model=en-US_NarrowbandModel"
```

**Note**: You will have to specify your `apikey` when making requests, additionally validate you are making requests to the correct service instance. For example, US South instances use the `stream.watsonplatform.net` hostname, while Washington DC instances use `gateway-wdc.watsonplatform.net`.


Example Output:

```bash
{
   "results": [
      {
         "alternatives": [
            {
               "confidence": 0.99,
               "transcript": "smoky fires lack flame and heat "
            }
         ],
         "final": true
      }
   ],
   "result_index": 0
}
dns_lookup: 0.063025
connect: 0.113092
appconnect: 0.229476
 pretransfer: 0.229533
redirect: 0.000000
starttransfer: 0.301009
total: 2.322544
```
## Speech-To-Text Curl Script: stt-curl.sh

This bash script will issue periodic requests to Watson Speech To Text (default period is every 2 seconds) and will print out cases where the response time exceeds a predefined threshold (default threshold is 2 seconds). The curl command generates all of the following:

- **HTTP Response Headers** - needed to access transaction IDs
- **JSON Response from STT** - which includes the STT ID (X-DP-Watson-Tran-ID and x-global-transaction-id)
- **Time Elements** - which provide details about where time was spent during the transaction

You will need to make the following two changes to the script before running it:
1. Be sure to set the API KEY to match your STT service instance.
1. Be sure to modify the STT service URL to match the URL associated with your service instance.

Note that this script first dumps all the results to a file (results.txt) and then uses awk to parse the last line of the file to get the total transaction time.

Here is an example of the response that is returned from this script:

```
time: 10/23/2019 16:16:23
HTTP/1.1 100 Continue
X-EdgeConnect-MidMile-RTT: 18
X-EdgeConnect-Origin-MEX-Latency: 24

HTTP/1.1 200 OK
Content-Type: application/json
session-name: JQUNRERVMOBJJDTA-en-US_NarrowbandModel
x-content-type-options: nosniff
content-disposition: inline; filename="result.json"
content-security-policy: default-src 'none'
x-xss-protection: 1
x-frame-options: DENY
x-global-transaction-id: e8ed299b921a1510a2d80e9459a6ad89
X-DP-Watson-Tran-ID: e8ed299b921a1510a2d80e9459a6ad89
Content-Length: 254
X-EdgeConnect-MidMile-RTT: 18
X-EdgeConnect-Origin-MEX-Latency: 4431
X-EdgeConnect-MidMile-RTT: 18
X-EdgeConnect-Origin-MEX-Latency: 24
Date: Wed, 23 Oct 2019 20:16:23 GMT
Connection: keep-alive

{
   "results": [
      {
         "alternatives": [
            {
               "confidence": 0.99,
               "transcript": "smoky fires lack flame and heat "
            }
         ],
         "final": true
      }
   ],
   "result_index": 0
}
dns_lookup: 0.004397
connect: 0.044387
appconnect: 0.829275
pretransfer: 0.829317
redirect: 0.000000
starttransfer: 0.970303
total: 5.402592
```
