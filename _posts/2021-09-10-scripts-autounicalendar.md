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

**autoUniCalendar**, available in [my own Github](https://github.com/Bimo99B9/autoUniCalendar), is a script which converts the Uniovi calendar into Google and Microsoft calendars by reading the HTTP requests to the web server, and parsing the calendar data into a CSV file readable by every calendar application.

![](/assets/images/scripts-autounicalendar/script.jpg)

## How it works

To understand how this script works, it is necessary to understand the process to display the calendar as usual in the SIES web page, so it is possible to replicate every step in a simple Python script.

First of all, the user logs into the intranet. This generates some cookies that the server needs to validate the requests later. Then, the data is processed and displayed in the browser. In the script, we could ask for the credentials and then retrieve the cookies, but we will ask for the cookies directly, so the user security can never be compromised, even though the script is more difficult to run.

# The HTTP Requests

To analyze the requests done to the server I used burpsuite and foxyproxy. I intercepted the main request to load the calendar in the SIES, and realized that it was a GET HTTP request which uses two relevant parameters, `JSESSIONID` and `oam.Flash.RENDERMAP.TOKEN`.

Those parameters are the cookie of the session of the user, which is a temporal identifier of its session, and from a security point of view, could only be used to impersonate the session, but we won't have the username and password exposed, and a Flash token necessary to process the calendar. Both can be obtained pressing `F12` in the web browser and navigating to `Application --> Storage --> Cookies`, and they will be the parameters of the script.

![](/assets/images/scripts-autounicalendar/cookies.jpg)

The response of this request retrieved the general information about the calendar web application. Then, some not important requests are done, and finally, a POST HTTP request sends us the raw events data of the calendar. This POST request needs a lot of parameters that I noticed that were contained in the response of the first GET request.

Therefore, the program needs to make the first request with the Session and Render tokens, and search in the response for the parameters `javax.faces.source`, `javax.faces.ViewState` and `javax.faces.source_SUBMIT`. This is a bit complex as they appear several times with different code structures in the HTTP response. 

The other parameters missing were the start and the end dates of the calendar events. Both are [UNIX timestamps](https://en.wikipedia.org/wiki/Unix_time), so we can modify them to make the server sends us the whole course data, and not only a single week.

The image below shows in the left the POST request with the payload built, and in the right the response generated with all the raw data retrieved.

![](/assets/images/scripts-autounicalendar/burp.jpg)

# Parsing the data

As we saw in the image of burpsuite, the data retrieved is not readable by any known program apart from the web calendar itself. Then, we have to treat the data and convert it in a CSV file with the fields that Outlook, Google Calendar and every calendar accepts. 

To do this, we can separate the events with the `{` separator, as each event is between curly braces. Then, if we split the data of each event with `,` and remove the empty spaces, we have a vector of eight positions with the eight parts of data of the event (Title, description...).

We still need to treat this data to make it perfectly fit in a CSV file. Then, many more procedures as the ones explained above are applied.

```python

titulo = re.findall('"([^"]*)"', title.split(':')[1])[0]
tmp = start.split(' ')[1].split('T')[0].removeprefix('"')
fechainicio = tmp.split('-')[2]+'/'+tmp.split('-')[1]+'/'+tmp.split('-')[0]
horainicio = start.split(' ')[1].split('T')[1].split('+')[0]
tmp = end.split(' ')[1].split('T')[0].removeprefix('"')
fechafin = tmp.split('-')[2]+'/'+tmp.split('-')[1]+'/'+tmp.split('-')[0]
horafin = end.split(' ')[1].split('T')[1].split('+')[0]
fechadealerta = fechainicio
horadealerta = str(int(start.split(' ')[1].split('T')[1].split('+')[0].split(':')[0]) - 1) + ':' + start.split(' ')[1].split('T')[1].split('+')[0].split(':')[1] + ':' + start.split(' ')[1].split('T')[1].split('+')[0].split(':')[2]
creador = "Universidad de Oviedo"
body = description.split('"')[3].replace(r'\n', '')
csvline = titulo+','+fechainicio+','+horainicio+','+fechafin+','+horafin+',FALSO,FALSO,'+fechadealerta+','+horadealerta+','+creador+',,,,,,'+body+',,,Normal,Falso,Normal,2\n'

```

# Importing the CSV

To end, the script deteles the raw data file, and writes each csv line in a `Calendar.CSV` file that can be imported in any known calendar application. To do this, you can open your calendar app, and select `import from a file`, and if asked, select `separated with commas`, to indicate that it is a CSV file.

## The script

The written script that makes the first GET request, extracts the cookies, makes the POST request, and converts the raw data into a CSV file, is available in [my own Github](https://github.com/Bimo99B9/autoUniCalendar) and below

```python

#!/usr/bin/python3
# coding: utf-8

import re
import requests
import sys
import urllib.parse
import os

if len(sys.argv) != 3:
    print("\n[!] Uso: python3 " + sys.argv[0] + " <JSESSIONID> <RENDERMAP.TOKEN>\n")
    sys.exit(1)

session = sys.argv[1]
rendermap = sys.argv[2]

print("[i] autoUniCalendar, a script to convert the Uniovi calendar to Google and Microsoft calendars.")
print("[i] Designed and programmed by Daniel López Gala from the University of Oviedo.")
print("[i] Visit Bimo99B9.github.io for more content.\n")
print("\n[*] The provided session is: " + session)
print("[*] The provided render token is: " + rendermap + "\n")


# print(session + rendermap)

def get_first_request(s, r):
    print("[@] Sending the first request...")
    url = 'https://sies.uniovi.es/serviciosacademicos/web/expedientes/calendario.xhtml'
    payload = {
        'JSESSIONID': s,
        'oam.Flash.RENDERMAP.TOKEN': r
    }
    r = requests.get(url, cookies=payload)
    print("[#] First request correctly finished.\n")
    return r.text


def extract_cookies(req):
    print("[@] Extracting the calendar parameters...")
    for line in req.split('\n'):
        if '<div id="j_id' in line:
            tmp = line.split('<')
            src = tmp[1]
            src2 = re.findall('"([^"]*)"', src)[0]
            source = urllib.parse.quote(src2)
            print("[*] javax.faces.source retrieved: " + source)
            break

    for line in req.split('\n'):
        if 'javax.faces.ViewState' in line:
            tmp = line
            st = tmp.split(' ')[12]
            viewst = re.findall('"([^"]*)"', st)[0]
            viewstate = urllib.parse.quote(viewst)
            print("[*] javax.faces.ViewState retrieved: " + viewstate)
            break

    for line in req.split('\n'):
        if 'action="/serviciosacademicos/web/expedientes/calendario.xhtml"' in line:
            tmp = line.split(' ')
            submit = re.findall('"([^"]*)"', tmp[3])[0]
            print("[*] javax.faces.source_SUBMIT retrieved: " + submit)
            break

    print("[#] Calendar parameters extracted.\n")

    return [source, viewstate, submit]


def post_second_request(s, r, ajax, source, view, start, end, submit):
    print("[@] Sending the calendar request...")

    url = 'https://sies.uniovi.es/serviciosacademicos/web/expedientes/calendario.xhtml'
    payload = {
        'JSESSIONID': s,
        'oam.Flash.RENDERMAP.TOKEN': r,
        'cookieconsent_status': 'dismiss'
    }
    stringstart = source + "_start"
    stringend = source + "_end"
    stringsubmit = submit + "_SUBMIT"
    headers = {'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'}

    print("[*] Creating the payload...")
    bodypayload = "javax.faces.partial.ajax=" + ajax + "&javax.faces.source=" + source + "&javax.faces.partial.execute=" + source + "&javax.faces.partial.render=" + source + "&" + source + "=" + source + "&" + stringstart + "=" + start + "&" + stringend + "=" + end + "&" + stringsubmit + "=1&javax.faces.ViewState=" + view

    r = requests.post(url, data=bodypayload, headers=headers, cookies=payload)
    print("[#] Calendar request correctly retrieved.\n")

    print("[@] Writing the raw calendar data into a .txt file...")
    f = open("raw.txt", "w")
    f.write(r.text)
    f.close()
    print("[#] File correctly written.\n")


def treatFile(file):

    print("[@] Creating the CSV file...")
    f = open(file, "r")
    g = open("Calendario.CSV", "w")
    g.write("Asunto,Fecha de comienzo,Comienzo,Fecha de finalización,Finalización,Todo el dí­a,Reminder on/off,Reminder Date,Reminder Time,Meeting Organizer,Required Attendees,Optional Attendees,Recursos de la reuniÃƒÂ³n,Billing Information,Categories,Description,Location,Mileage,Priority,Private,Sensitivity,Show time as\n")
    data = f.read()
    text = data.split('<')
    calendar = text[5]
    events = calendar.split('{')
    del events[0:2]
    print("[*] Parsing the data...")
    for event in events:
        res = []
        for ele in event.split(','):
            if ele.strip():
                res.append(ele)
        id = res[0]
        title = res[1]
        start = res[2]
        end = res[3]
        allday = res[4]
        editable = res[5]
        className = res[6]
        description = res[7]
        
        titulo = re.findall('"([^"]*)"', title.split(':')[1])[0]
        tmp = start.split(' ')[1].split('T')[0].removeprefix('"')
        fechainicio = tmp.split('-')[2]+'/'+tmp.split('-')[1]+'/'+tmp.split('-')[0]
        horainicio = start.split(' ')[1].split('T')[1].split('+')[0]
        tmp = end.split(' ')[1].split('T')[0].removeprefix('"')
        fechafin = tmp.split('-')[2]+'/'+tmp.split('-')[1]+'/'+tmp.split('-')[0]
        horafin = end.split(' ')[1].split('T')[1].split('+')[0]
        fechadealerta = fechainicio
        horadealerta = str(int(start.split(' ')[1].split('T')[1].split('+')[0].split(':')[0]) - 1) + ':' + start.split(' ')[1].split('T')[1].split('+')[0].split(':')[1] + ':' + start.split(' ')[1].split('T')[1].split('+')[0].split(':')[2]
        creador = "Universidad de Oviedo"
        body = description.split('"')[3].replace(r'\n', '')
        csvline = titulo+','+fechainicio+','+horainicio+','+fechafin+','+horafin+',FALSO,FALSO,'+fechadealerta+','+horadealerta+','+creador+',,,,,,'+body+',,,Normal,Falso,Normal,2\n'
        g.write(csvline)
        
    print("[*] Events correctly written in the CSV file.")
    f.close()
    g.close()
    print("[*] Removing raw .txt file...")
    os.remove("raw.txt")
    print("\n[#] Calendar generated. You can now import it in Outlook or Google Calendar selecting 'import from file' and providing the CSV file generated.\n")


first_request = get_first_request(session, rendermap)
cookies = extract_cookies(first_request)
post_second_request(session, rendermap, "true", cookies[0], cookies[1], "1630886400000", "1652054400000", cookies[2])
treatFile("raw.txt")

```

## How to use it

To use the script, it is mandatory to have installed the `requests` library of Python, which can be installed in Windows with `python3 -m pip install requests` and in Linux with `pip install requests`.

Then, to run it, in a console opened in the folder where the script is located, the will type `python3 autoUniCalendar.py <JSessionID> <RenderToken>`.

