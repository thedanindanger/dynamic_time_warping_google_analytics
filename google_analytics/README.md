
# Google Analytics Event Parsing Script

## Disclaimer
Modified from code developed by api.nickm@gmail.com (Nick Mihailovski) under the Apache License
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.#Will need to set variables here
Begin by registering app on google api services

## Introduction

The following code pulls Google Analytics events from a given GA account. It iterates through the provided date ranges in blocks so as to minimize sampling. With this code we can pull large amounts of detailed GA data with relatively low effort.

It has been modified from the original source to include more automation in date parsing, more robust error handling, and simplified value assignment

## Getting Started

To use the GA API, you most have a project registered with Google and a JSON file with your key information:

> https://console.developers.google.com/flows/enableapi?apiid=analytics

* Create New Project
* Go to 'Credentials' on lefthand side
* Select Oauth authentication in either blue text near center screen (Should be an option to skip configuration)
* Name the app whatever you like
* Select 'Other' for application type
* Name the application type whatever you want as well
* Download the client secret data using the icon on the far right after the credentials display
* Replace provided 'client_secrets.json' with entirety of downloaded file

If needed you may install google analytic python libraries via

```
pip install --upgrade google-api-python-client
```

To make it easier, you are able to explore query parameters here for testing accounting access and verifying results

> https://ga-dev-tools.appspot.com/query-explorer/

This comes in handy if you need to select other parameters than the one given in the script

## Loading Libraries and House Keeping


```python
import argparse
import sys
import csv
import string

from apiclient.errors import HttpError
from apiclient import sample_tools
from oauth2client.client import AccessTokenRefreshError

import time
from datetime import datetime, timedelta, date

class SampledDataError(Exception): pass
```

## Variable Assignment

Set Variables: confirm profile_ids, date_ranges, and file_path are set

Also be sure the client_secret.json file is present in same directory as python script



```python
profile_ids = {'yourProfileNameHere':  'profileIDNumberAsStringHere'}

'''Date Settings'''
#This allows for date splitting in the future, produces string with yesterday's date
endDate = date.today() - timedelta(days=1) #sets date to yesterday
startDate = date(2016,1,1) #Year,Month,Day
daysbetween = 10

#Specify where the file should write, automatically generates name based on user id

file_path = '../data/'

__author__ = 'Daniel Smith'

```

## Creating a Date Tuple to Iterate Over

Using the current date and the date defined above, the follwoing script create a tuple of dates. Each tuple value contains a start and end date.


```python
today = date.today()
timestamp = (today.strftime("%Y-%m-%d"))

def make_date_tuple (startDate, endDate, daysbetween):
    dateagg = startDate
    dateTuple = []
    #days = (endDate-startDate).days
    while ((dateagg + timedelta(days=daysbetween)) < endDate):
        d1 = dateagg
        d2 = (dateagg + timedelta(days=daysbetween))
        d1str = (d1.strftime("%Y-%m-%d"))
        d2str = (d2.strftime("%Y-%m-%d"))
        dateTuple.append((d1str,d2str))
        dateagg = (d2 + timedelta(days=1))
    dateTuple.append((dateagg.strftime("%Y-%m-%d"),endDate.strftime("%Y-%m-%d")))
    return dateTuple

date_ranges = make_date_tuple(startDate,endDate, daysbetween)
```

## Connecting to GA

Here is where we connect to the GA API given the profile populated above. The bulk of the code below is error handling this connection.



```python
def main(argv):
  # Authenticate and construct service.
  service, flags = sample_tools.init(
      argv, 'analytics', 'v3', __doc__, __file__,
      scope='https://www.googleapis.com/auth/analytics.readonly')

  # Try to make a request to the API. Print the results or handle errors.
  try:
    profile_id = profile_ids[profile]
    if not profile_id:
      print 'Could not find a valid profile for this user.'
    else:
     for n in range(0, 10):
      try:
       for start_date, end_date in date_ranges:
        limit = ga_query(service, profile_id, 0,
                                 start_date, end_date).get('totalResults')
        for pag_index in xrange(0, limit, 10000):
          event_results = ga_query(service, profile_id, pag_index,
                                     start_date, end_date)
          if event_results.get('containsSampledData'):
            
            raise SampledDataError
          print_results(event_results, pag_index, start_date, end_date)
      except HttpError, error:
       if error.resp.reason in ['userRateLimitExceeded', 'quotaExceeded',
                               'internalServerError', 'backendError']:
          print 'Hit an API error due to rate limits, sleeping and re-trying'
          time.sleep((2 ** n) + random.random())
       else:
       	  break

  except TypeError, error:    
    # Handle errors in constructing a query.
    print ('There was an error in constructing your query : %s' % error)

  except HttpError, error:
    # Handle API errors.
    print ('Arg, there was an API error : %s : %s' %
           (error.resp.status, error._get_reason()))

  except AccessTokenRefreshError:
    # Handle Auth errors.
    print ('The credentials have been revoked or expired, please re-run '
           'the application to re-authorize')
  
  except SampledDataError:
    # force an error if ever a query returns data that is sampled!
    print ('Error: Query contains sampled data!')
```

## GA Query

Here's the script that defines what data we are going to pull now that we're connected to GA. Its pretty self explanatory, but if you need a reminder on the syntax, use the GA Query Tool linked above and strip the parameter arguements off the GA URL, they syntax is exactly the same


```python
def ga_query(service, profile_id, pag_index, start_date, end_date):

  return service.data().ga().get(
      ids='ga:' + profile_id,
      start_date=start_date,
      end_date=end_date,
      #unique events only counts one event per session (i.e. visit) better for evaluating WTB clicks
      metrics='ga:users, ga:sessions', 
      dimensions='ga:year,ga:month,ga:region,ga:city',
      #sort='-ga:totalEvents',
      #filters='ga:campaign==%s,ga:campaign==%s' %(oldCampaign, newCampaign),
      samplingLevel='HIGHER_PRECISION',
      start_index=str(pag_index+1),
      max_results=str(pag_index+10000)).execute()
```

# Writing Data

Our last function is boiler plate csv writitng, but it does include some console logging for progress monitoring


```python
def print_results(results, pag_index, start_date, end_date):

  if pag_index == 0:
    if (start_date, end_date) == date_ranges[0]:
      print 'Profile Name: %s' % results.get('profileInfo').get('profileName')
      columnHeaders = results.get('columnHeaders')
      cleanHeaders = [str(h['name']) for h in columnHeaders]
      writer.writerow(cleanHeaders)
    print 'Now pulling data from %s to %s.' %(start_date, end_date)



  # Print data table.
  if results.get('rows', []):
    for row in results.get('rows'):
      for i in range(len(row)):
        old, new = row[i], str()
        for s in old:
          new += s if s in string.printable else ''
        row[i] = new
      writer.writerow(row)

  else:
    print 'No Rows Found'

  limit = results.get('totalResults')
  print pag_index, 'of about', int(round(limit, -4)), 'rows.'
  return None
```


```python
for profile in sorted(profile_ids):
  path = file_path #replace with path to your folder where csv file with data will be written
  filename = 'google_analytics_weekly_event_data_%s.csv' #replace with your filename. Note %s is a placeholder variable and the profile name you specified on row 162 will be written here
  with open(path + filename %(profile.lower()), 'wt') as f:
    writer = csv.writer(f, lineterminator='\n')
    if __name__ == '__main__': main(sys.argv)
  print "Profile done. Next profile..."

print "All profiles done."
```

## Closing Thoughts

I probably should have mentioned this script needs to be run in the terminal / command line to work. You can't run it from Jupyter :)
