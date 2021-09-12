---
layout: single
title: Scripts - autoUniCalendar
excerpt: "autoUniCalendar is a script which converts the Uniovi calendar into Google and Microsoft calendars by reading the HTTP requests to the web server, and parsing the calendar data into a CSV file readable by every calendar application."
date: 2021-09-10
classes: wide
header:
  teaser: /assets/images/scripts-autounicalendar/logo.png
  teaser_home_page: true
  icon: /assets/images/script.png
categories:
  - scripts
  - python
tags:  
  - uniovi
  - productivity
  - github
  - scripting
  - programming
---

![](/assets/images/scripts-autounicalendar/logo.png)

## autoUniCalendar

**autoUniCalendar**, first posted on [my Twitter](https://twitter.com/Bimo99B9/status/1436034252462755841?s=20), and available in [my Github](https://github.com/Bimo99B9/autoUniCalendar), is a script which converts the Uniovi calendar into Google and Microsoft calendars by reading the HTTP requests to the web server, and parsing the calendar data into a CSV file readable by every calendar application.

![](/assets/images/scripts-autounicalendar/cat.jpg)

## How it works

To understand how this script works, it is necessary to understand the process to display the calendar as usual in the SIES web page, so it is possible to replicate every step in a simple Python script.

First of all, the user logs into the intranet. This generates some cookies that the server needs to validate the requests later. Then, the data is processed and displayed in the browser. In the script, we could ask for the credentials and then retrieve the cookies, but we will ask for the cookies directly, so the user security can never be compromised, even though the script is more difficult to run.

![](/assets/images/scripts-autounicalendar/script.jpg)

# The HTTP Requests

To analyze the requests done to the server I used burpsuite and foxyproxy. I intercepted the main request to load the calendar in the SIES, and realized that it was a GET HTTP request which uses two relevant parameters, `JSESSIONID` and `oam.Flash.RENDERMAP.TOKEN`.

Those parameters are the cookie of the session of the user, which is a temporal identifier of its session, and from a security point of view, could only be used to impersonate the session, but we won't have the username and password exposed, and a Flash token necessary to process the calendar. Both can be obtained pressing `F12` in the web browser and navigating to `Application --> Storage --> Cookies`, and they will be the parameters of the script.

![](/assets/images/scripts-autounicalendar/cookies.jpg)

The response of this request retrieved the general information about the calendar web application. Then, some not important requests are done, and finally, a POST HTTP request sends us the raw events data of the calendar. This POST request needs a lot of parameters that I noticed that were contained in the response of the first GET request.

Therefore, the program needs to make the first request with the Session and Render tokens, and search in the response for the parameters `javax.faces.source`, `javax.faces.ViewState` and `javax.faces.source_SUBMIT`. This is a bit complex as they appear several times with different code structures in the HTTP response. 

The other parameters missing were the start and the end dates of the calendar events. Both are [UNIX timestamps](https://en.wikipedia.org/wiki/Unix_time), so we can modify them to make the server send us the whole course data, and not only a single week.

The image below shows in the left the POST request with the payload built, and in the right the response generated with all the raw data retrieved.

![](/assets/images/scripts-autounicalendar/burp.jpg)

# Parsing the data

As we saw in the image of burpsuite, the data retrieved is not readable by any known program apart from the web calendar itself. Then, we have to treat the data and convert it in a CSV file with the fields that Outlook, Google Calendar and every calendar accepts. 

To do this, we can separate the events with the `{` separator, as each event is between curly braces. Then, if we split the data of each event with `,` and remove the empty spaces, we have a vector of eight positions with the eight parts of data of the event (Title, description...).

We still need to treat this data to make it perfectly fit in a CSV file. Then, many more procedures as the ones explained above are applied.

```python
title_csv = re.findall('"([^"]*)"', title.split(':')[1])[0]
start_date = start.split(' ')[1].split('T')[0].split('"')[1]
start_date_csv = start_date.split('-')[2]+'/'+start_date.split('-')[1]+'/'+start_date.split('-')[0]
start_hour = start.split(' ')[1].split('T')[1].split('+')[0]
end_date = end.split(' ')[1].split('T')[0].removeprefix('"')
end_date_csv = end_date.split('-')[2]+'/'+end_date.split('-')[1]+'/'+end_date.split('-')[0]
end_hour = end.split(' ')[1].split('T')[1].split('+')[0]
alert_date = start_date_csv
alert_hour = str(int(start.split(' ')[1].split('T')[1].split('+')[0].split(':')[0]) - 1) + ':' + start.split(' ')[1].split('T')[1].split('+')[0].split(':')[1] + ':' + start.split(' ')[1].split('T')[1].split('+')[0].split(':')[2]
event_creator = "Universidad de Oviedo"
body = description.split('"')[3].replace(r'\n', '')
```


# Importing the CSV

To end, the script deteles the raw data file, and writes each csv line in a `Calendar.CSV` file that can be imported in any known calendar application. To do this, you can open your calendar app, and select `import from a file`, and if asked, select `separated with commas`, to indicate that it is a CSV file.

## The script

The written script that makes the first GET request, extracts the cookies, makes the POST request, and converts the raw data into a CSV file, is available in [my own Github](https://github.com/Bimo99B9/autoUniCalendar) and below.

```python
#!/usr/bin/python3
# coding: utf-8

import re
import requests
import sys
import urllib.parse
import os

# Check if the required arguments have been provided, and indicate the use of the script.
if len(sys.argv) != 3:
    print("\n[!] Uso: python3 " + sys.argv[0] + " <JSESSIONID> <RENDERMAP.TOKEN>\n")
    sys.exit(1)

# Declare global variables.
url = 'https://sies.uniovi.es/serviciosacademicos/web/expedientes/calendario.xhtml'
session = sys.argv[1]
render_map = sys.argv[2]

# Script information.
print("[i] autoUniCalendar, a script which converts the Uniovi calendar into Google and Microsoft calendars.")
print("[i] Designed and programmed by Daniel López Gala from the University of Oviedo.")
print("[i] Visit Bimo99B9.github.io for more content.\n")
print(f"[*] The provided session is: {session}")
print(f"[*] The provided render token is: {render_map}")

# Function to send the first GET HTTP request using the tokens provided.
def get_first_request(session_token, render_token):

    print("[@] Sending the first request...")

    # Cookies payload of the HTTP request.
    payload = {
        'JSESSIONID': session_token,
        'oam.Flash.RENDERMAP.TOKEN': render_token
    }

    r = requests.get(url, cookies=payload)
    print("[#] First request correctly finished.\n")
    # The function returns the server response to use it later.
    return r.text

# Function to extract the cookies necessary to make the POST request, from the server response of the first request.
def extract_cookies(get_response):
    print("[@] Extracting the calendar parameters...")

    # Iterate the response lines to search the cookies, and save them in variables.
    # To-do: Optimize search algorithm.
    for line in get_response.split('\n'):
        if '<div id="j_id' in line:
            source = urllib.parse.quote(re.findall('"([^"]*)"', line.split('<')[1])[0])
            print(f"[*] javax.faces.source retrieved: {source}")
            break

    for line in get_response.split('\n'):
        if 'javax.faces.ViewState' in line:
            viewstate = urllib.parse.quote(re.findall('"([^"]*)"', line.split(' ')[12])[0])
            print(f"[*] javax.faces.ViewState retrieved: {viewstate}")
            break

    for line in get_response.split('\n'):
        if 'action="/serviciosacademicos/web/expedientes/calendario.xhtml"' in line:
            submit = re.findall('"([^"]*)"', line.split(' ')[3])[0]
            print(f"[*] javax.faces.source_SUBMIT retrieved: {submit}")
            break

    print("[#] Calendar parameters extracted.\n")
    # The function returns a list that contains the extracted parameters.
    return [source, viewstate, submit]

# Function that sends the HTTP POST request to the server and retrieves the raw data of the calendar.
def post_second_request(session_token, render_token, ajax, source, view, start, end, submit):

    print("[@] Sending the calendar request...")
    
    # Cookies of the request.
    payload = {
        'JSESSIONID': session_token,
        'oam.Flash.RENDERMAP.TOKEN': render_token,
        'cookieconsent_status': 'dismiss'
    }

    # Define variables of the request.
    string_start = source + "_start"
    string_end = source + "_end"
    string_submit = submit + "_SUBMIT"

    # Creating the body with the parameters extracted before, with the syntax required by the server.
    print("[*] Creating the payload...")
    body_payload = f"javax.faces.partial.ajax={ajax}&javax.faces.source={source}&javax.faces.partial.execute={source}&javax.faces.partial.render={source}&{source}={source}&{string_start}={start}&{string_end}={end}&{string_submit}=1&javax.faces.ViewState={view}"

    # Send the POST request. 
    r = requests.post(url, data=body_payload, headers={'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'}, cookies=payload)
    print("[#] Calendar request correctly retrieved.\n")

    # Write the raw response into a temporary file.
    print("[@] Writing the raw calendar data into a .txt file...")
    f = open("raw.txt", "w")
    f.write(r.text)
    f.close()
    print("[#] File correctly written.\n")

# Function that creates a CSV file readable by the applications, from the raw data previously retrieved.
def create_csv(file):

    print("[@] Creating the CSV file...")
    
    # Create the file.
    f = open(file, "r")
    g = open("Calendario.CSV", "w")

    # Write the headers in the first line.
    g.write("Asunto,Fecha de comienzo,Comienzo,Fecha de finalización,Finalización,Todo el dí­a,Reminder on/off,Reminder Date,Reminder Time,Meeting Organizer,Required Attendees,Optional Attendees,Recursos de la reuniÃƒÂ³n,Billing Information,Categories,Description,Location,Mileage,Priority,Private,Sensitivity,Show time as\n")
    
    # Separate the events from its XML context.
    text = f.read().split('<')
    events = text[5].split('{')
    del events[0:2]

    # Each field of the event is separated by commas.
    print("[*] Parsing the data...")
    for event in events:
        data = []
        for field in event.split(','):
            # Remove empty fields.
            if field.strip():
                data.append(field)
        # Save in variables the fields needed to build the CSV line of the event.
        title = data[1]
        start = data[2]
        end = data[3]
        description = data[7]
        
        # Make the necessary strings transformations to adapts the raw field data into a CSV readable file.
        title_csv = re.findall('"([^"]*)"', title.split(':')[1])[0]
        start_date = start.split(' ')[1].split('T')[0].split('"')[1]
        start_date_csv = start_date.split('-')[2]+'/'+start_date.split('-')[1]+'/'+start_date.split('-')[0]
        start_hour = start.split(' ')[1].split('T')[1].split('+')[0]
        end_date = end.split(' ')[1].split('T')[0].removeprefix('"')
        end_date_csv = end_date.split('-')[2]+'/'+end_date.split('-')[1]+'/'+end_date.split('-')[0]
        end_hour = end.split(' ')[1].split('T')[1].split('+')[0]
        alert_date = start_date_csv
        alert_hour = str(int(start.split(' ')[1].split('T')[1].split('+')[0].split(':')[0]) - 1) + ':' + start.split(' ')[1].split('T')[1].split('+')[0].split(':')[1] + ':' + start.split(' ')[1].split('T')[1].split('+')[0].split(':')[2]
        event_creator = "Universidad de Oviedo"
        body = description.split('"')[3].replace(r'\n', '')
        # Write all the fields into a single line, and append it to the file.
        csv_line = f"{title_csv},{start_date_csv},{start_hour},{end_date_csv},{end_hour},FALSO,FALSO,{alert_date},{alert_hour},{event_creator},,,,,,{body},,,Normal,Falso,Normal,2\n"
        g.write(csv_line)
        
    print("[*] Events correctly written in the CSV file.")
    f.close()
    g.close()
    print("[*] Removing raw .txt file...")
    os.remove("raw.txt")
    print("\n[#] Calendar generated. You can now import it in Outlook or Google Calendar selecting 'import from file' and providing the CSV file generated.\n")


first_request = get_first_request(session, render_map)
cookies = extract_cookies(first_request)
post_second_request(session, render_map, "true", cookies[0], cookies[1], "1630886400000", "1652054400000", cookies[2])
create_csv("raw.txt")
```

## How to use it

To use the script, it is mandatory to have installed the `requests` library of Python, which can be installed in Windows with `python3 -m pip install requests` and in Linux with `pip install requests`.

Then, to run it, in a console opened in the folder where the script is located, the user will type `python3 autoUniCalendar.py <JSessionID> <RenderToken>`.

