# Wekan Scheduler

Pythonic card scheduler for [Wekan](https://wekan.github.io/).


## About

The purpose of *wekan-scheduler* is to create cards regulary for repeating
tasks. It uses [pycron](https://github.com/kipe/pycron) for schedule
configuration and uses the
[wekan-python-api-client](https://github.com/wekan/wekan-python-api-client) to
access the [Wekan REST API](https://wekan.github.io/api/v3.00/).


## Features

- create cards by a cron-like schedule
- suppress card creation based on iCal calendar files (i.e. on holidays)
- configurable card properties:
  - title
  - description
  - color
  - labels
  - members
  - received, start, due, end times (with dynamic calculation)
  - checklists

## Configuration

You should create a dedicated user in *Wekan* to be used to access the *Wekan
REST API* in the `PASSWORD` auth backend (`LDAP` seems not to work). Add the
user to the boards where it should create cards.

There are three configuration files required in the `settings` directory:

- **\_\_init\_\_.py**
  should left empty, required to make the config a python package
- **config.py**
  contains the base configuration to access the *Wekan REST API*:
  ```python
  CONFIG = {
    # URL for Wekan
    'api_url': "https://localhost/kanban",

    # REST API auth headers
    'api_auth': {"username": "api", "password": "xxxxxxxx"},

    # Verify X.509 certificate for api_url?
    'api_verify': True,

    # Sleep for N seconds between schedules (should be at least 60s)
    'sleep': 300,
  }
  ```
- **schedules.py**
  contains your custom schedules:
  ```python
  from datetime import datetime as dt
  from datetime import timedelta as td
  import pytz

  SCHEDULES = [
    # daily work card
    {
        # cron-like schedule (pycron)
        'schedule': '0 8 * * 1-5',

        # do *not* create a card if a calendar event is found
        # in a iCal file in the ics/ subdirectory
        #
        # to enable this feature the value needs to be a dict,
        # if filename or event filters are unset they match
        # for any calendar event
        #
        # fn: check .ics filename (regex)
        # event: check EVENT name (regex)
        #
        # This will suppress card creation by calendar events
        # with an name containing the word 'Holiday' found in
        # any iCal file:
        #
        # 'ics': {
        #    'event': 'Holiday',
        # },

        # target board and list
        'board': '<board-id>',
        'list': '<list-id>',

        # required fields
        'card': {
            'title': 'check for donuts',
        },

        # more card details
        # https://wekan.github.io/api/v3.00/#put_board_list_card
        'details': {
            'description': 'first-come, first-served',
            'color': 'pink',
            'labelIds': ['<label-id>'],
            # set due time at 17 o'clock tomorrow
            #'dueAt': lambda: (dt.now() + td(days=1)).replace(hour=17, minute=0, second=0, microsecond=0).astimezone(pytz.utc).isoformat(),
            'members': ['<user-id>'],
        },

        # add checklists and checklist items
        # (requires Wekan 3.46+)
        'checklists': {
          'Steps': ['buy', 'eat'],
        },
    },
  ]
  ```
  If the value of a `details` dictonary item is a *callable* it is used to build
  the actual value of the item. This could be used to build dynamic start, due
  or end times.

There is a `get-ids.py` helper script to dump the various IDs of your Wekan
instance. This helps to find the required IDs for your schedules:

```
{'boards': {'4b5BmHE2CL8wt8brR': {'labels': {'2kRvzr': 'gray[crumbly]',
                                             'jdiYNd': 'plum[goo]',
                                             'xhAdgq': 'lime[tasty]'},
                                  'lists': {'zzujS93kd8982k30y': 'Backlog',
                                            'loxc9Dll,masd300a': 'Eating',
                                            'qosLk34SDKsmsksd3': 'Done',
                                            'lasd93lkdaskSAKsl': 'Problem'},
                                  'title:': 'Donut Fighters'}},
 'users': [{'_id': 'Ksd34pKsW23wFg9Sx', 'username': 'admin'},
           {'_id': 'lS9lkasd3mAusdkSK', 'username': 'api'}]}
```


## Deploy

The recommended way to run *wekan-scheduler* is to use the provided auto-build
docker images from [Docker
Hub](https://cloud.docker.com/u/liske/repository/docker/liske/wekan-scheduler).
Run it with *docker-compose*:

```yaml
version: '3'

services:
  scheduler:
    image: liske/wekan-scheduler:0.7
    restart: always
    volumes:
      - ./settings:/app/wekan-scheduler/settings:ro
      - ./ics:/app/wekan-scheduler/ics:ro
```

If you are running *Wekan* on *Docker* you may add *wekan-scheduler* to your
existing `docker-compose.yml`:

```yaml
  # ...
  scheduler:
    image: liske/wekan-scheduler:0.7
    restart: always
    volumes:
      - ./scheduler/settings:/app/wekan-scheduler/settings:ro
      - ./scheduler/ics:/app/wekan-scheduler/ics:ro
  depends_on:
    - wekan
  # ...
```

Remember to use the docker's service internal name (i.e. `'api_url':
'http://wekan-app'`) in `config.py` to access wekan if deployed by the same
compose file.
