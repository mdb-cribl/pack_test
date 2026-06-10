# Qualys Detections
----

Using the provided python script and event breaker (see below), collect Qualys detection data, break it into 1 event per host (EB), then again into 1 event per detection. The XML will be converted to JSON. Optionally, send as JSON (Elastic, etc), or convert it into stringified JSON in _raw (ie, for Splunk).

## Requirements Section

Ensure that you have met the following requirements:

* Set-up an API account in Qualys. We'll use the username and password below.
* Ensure all worker nodes have python 3 installed, along with the `requests` module

## Installation

* Install the Event Breaker config included in JSON below
   * Processing -> Knowledge -> Event Breaker Rules -> Add Ruleset
   * Edit as JSON
   * Paste and save
* Create a new Scripted Collector (Data -> Sources -> Collectors Script)
   * Name it `qualys-detections`
   * Paste the python script included below into *both* Discover and Collect script areas
   * **Shell**: `/usr/bin/python3` (OR the path to your installed binary of python 3)
   * **Environment variables**:
      * **CRIBL_earliest**: `C.Time.strftime(Date.now()/1000 - 1800,"%Y-%m-%dT%H:%M:00Z")`
      * **CRIBL_username**: `your-api-username`
      * **CRIBL_password**: `your-api-password`
      * **CRIBL_max**: `3000` *(the number of hosts 1 thread will run for)*
   * **Schedule**: Run every 30 minutes: `*/30 * * * *`
   * **Event Breaker**: `qualys` (the EB you installed above)
   * Save
   * Create a Route or Quick Connect, pointing data from the Scripted Collector to this Pack
   * Adjust the pipeline as required for your output style
      * 2 options are provided by default: Splunk (strigified JSON in _raw), or JSON

## Release Notes

### Version 1.0.1 - 2026-02-27
- Fixed date-time format (in env vars, previously missing Z)
- Fixed login, logout and run_detections functions to use a dict so paylaod is properly encoded in the case of special characters

### Version 1.0.0 - 2023-07-26
Initial release

## Contributing to the Pack
Contact the author to contribute improvements and fixes

## Contact
To contact us please email <jrust@cribl.io>


## License
This Pack uses the following license: [`Apache 2.0`](https://github.com/criblio/appscope/blob/master/LICENSE).

---

### Artifacts

#### Event Breaker

```
{
  "id": "qualys",
  "lib": "custom",
  "minRawLength": 256,
  "rules": [
    {
      "condition": "_raw.includes('<\\?xml')",
      "type": "regex",
      "timestampAnchorRegex": "/DATETIME>/",
      "timestamp": {
        "type": "auto",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 1000000,
      "disabled": false,
      "parserEnabled": false,
      "eventBreakerRegex": "/(?=<HOST>)/",
      "name": "detection-list",
      "fields": [
        {
          "name": "_raw",
          "value": "_raw.replace(/<\\/HOST>.*/s,'</HOST>')"
        },
        {
          "name": "source",
          "value": "'qualys'"
        }
      ]
    }
  ]
}
```
---
### Python Script

```
#!/usr/bin/python3

import sys
import requests
import re
import os
from pprint import pprint
from time import time
from datetime import datetime

debug = True
action = ''
APIURL = "https://qualysapi.qg2.apps.qualys.com"

try:
    API_USER = os.environ['CRIBL_username']
    API_PWD = os.environ['CRIBL_password']
    EARLIEST = os.environ['CRIBL_earliest']
    MAX_RUN_COUNT = int(os.environ['CRIBL_max'])
except:
    print("CRIBL_username, CRIBL_password, CRIBL_max and CRIBL_earliest must be defined environment vars")
    sys.exit(1)

################################
# if the collect arg var exists, we already disco'd
if 'CRIBL_COLLECT_ARG' in os.environ:
    ids = os.environ['CRIBL_COLLECT_ARG'].split(' ')
    id_count = len(ids)
    ids = ','.join(str(item) for item in ids)
    action = 'collect'
else:
    action = 'disco'

################################
def login():
    url = APIURL + '/api/2.0/fo/session/'
    
    payload = dict(action="login", username=API_USER, password=API_PWD)
    headers = {
        'X-Requested-With': 'Cribl Stream'
    }
    response = requests.post(url, headers=headers, data=payload)
    
    if response.status_code == 200:
        return response.cookies['QualysSession']
    else:
        print('Error:', response.status_code)
        sys.exit(1)

################################
def logout(cookie):
    url = APIURL + '/api/2.0/fo/session/'
    payload = dict(action="logout")
    headers = {
        'X-Requested-With': 'Cribl Stream',
        'Cookie': 'QualysSession='+cookie,
        'Cookie2': '$Version="1"'
    }
        
    response = requests.post(url, headers=headers, data=payload)
    
    if response.status_code == 200:
        return True
    else:
        print('Logout Error:', response.status_code)
        pprint(vars(response.request))
        sys.exit(1)

################################
def list_hosts(cookie):
    url = APIURL + '/api/2.0/fo/asset/host/?action=list&truncation_limit=0&vm_processed_after=' + EARLIEST

    headers = {
        'X-Requested-With': 'Cribl Stream',
        'Content-Type': 'application/x-www-form-urlencoded',
        'Cookie': 'QualysSession='+cookie,
        'Cookie2': '$Version="1"'
    }
    
    response = requests.get(url, headers=headers)
    
    try:
        if response.status_code == 200:
            return(response.text)
        else:
            print('list_hosts failed with code:', response.status_code)
            logout(cookie)
            sys.exit(1)
    except Exception as e:
        print("list_hosts ERROR2: " + str(e))
        print("logging out")
        logout(cookie)
        sys.exit()

################################
def run_detections(cookie):
    url = APIURL + '/api/2.0/fo/asset/host/vm/detection/'

    payload = dict(output_format="XML",action="list",truncation_limit="0",detection_updated_since=EARLIEST,ids=ids)
    length = len(payload)
    headers = {
        'X-Requested-With': 'Cribl Stream',
        'Cookie': 'QualysSession='+cookie,
        'Cookie2': '$Version="1"'
    }
    
    response = requests.post(url, headers=headers, data=payload)
    
    #if debug:
    #    print("request")
    #    pprint(vars(response.request))
    
    try:
        if response.status_code == 200:
            return(response.text)
        else:
            print('run_detection failed with code:', response.status_code)
            #logout(cookie)
            sys.exit(1)
    except Exception as e:
        print("run_detection ERROR: " + str(e))
        sys.exit()

################################
# throw some debug info into a file
if debug:
    output_file = '/tmp/qualys-debug-' + str(time()) + '.txt'

    with open(output_file, 'w') as file:
        if action == 'collect':
            file.write('collect\ntimestamp: ' + EARLIEST)
        else:
            file.write('disco\ntimestamp: ' + EARLIEST)
        
        # Dump environment variables
        file.write('\nEnv:\n')
        for key, value in os.environ.items():
            if key.startswith('CRIBL'):
                file.write(f'{key}: {value}\n')

################################
# first get logged in
cookie = login()
#if debug:
#    print("got cookie %",cookie)

# get a list of ids if action is list
if action == 'disco':
    # get the list of host ids
    xml = list_hosts(cookie)

    # print the list of host ids in MAX_RUN_COUNT chunks
    try:
        ids = re.findall(r'<ID>(.*?)</ID>', xml)
        for i in range(0,len(ids),MAX_RUN_COUNT):
            print(*ids[i:i+MAX_RUN_COUNT])

    except Exception as e:
        print("died at id extract")
        pprint(vars(xml))

# or we want detections
elif action == 'collect':
    try:
        xml = run_detections(cookie)
        print(xml)
    except Exception as e:
        print("died while getting detections")
        pprint(vars(xml))

    # debugging
    if debug:
        output_file = '/tmp/qualys-debug-out-' + str(time()) + '.txt'
        with open(output_file, 'w') as file:
            file.write(xml)

logout(cookie)

```