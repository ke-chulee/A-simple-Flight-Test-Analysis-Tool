# A-simple-Flight-Test-Analysis-Tool

To build a simple Flight-Test Analysis Tool using the Dash-application Python framework that displays the strip charts in real-time, and analyzes recorded signals through interactive-graphing libraries.

## Dependencies
```
import math
import pandas as pd
import numpy as np
from plotly.subplots import make_subplots
import plotly.graph_objects as go
from scipy.fft import fft, fftfreq, fftshift
from dash import Dash, dcc, html, Input, Output, callback, State
```

## Executing program
### Load provided data.
```
dataset = pd.read_csv("roll_attitude_frequency_sweep.csv")
```
### Build page_content that uses URL to control recording and post-procssing
```
@callback(
    Output('page_content', 'children'),
    Input('url', 'pathname')
    )
def display_page(pathname):
    if pathname == '/page-1':
        return realtime_layout
    elif pathname == '/page-2':
        return postprocess_layout
    else:
        return home_page
```
Home page: 
```
home_page = html.Div([
    dcc.Link('Start Recording', href='/page-1'),
])
```
#### Real Time data page: 
Allow user-specified updata display rate through dcc.Input; Support unit transformation through dcc.RadioItems

```
realtime_layout = html.Div([
    html.H1('Real Time Data'),
    html.Div([
        html.Label('Input interval frame [ms]'),
        dcc.Input(
            type='number',
            id='intervalRateMs',
            value='1000',
            min=1, 
            max=200000, 
            step=1000,
            ),
        html.Label('Input history time window'),
        dcc.Input(
            type='number',
            id='timeHistoryToDisplay',
            value='30',
            min=1, 
            max=200, 
            step=5,
            )
    ],
    ),

    html.Div([
        dcc.RadioItems(
            options=[
            {'label':'Radius', 'value':'rad'},
            {'label':'Degree', 'value':'deg'}
            ],
            value='deg',
            id='Chart_unit',
            inline=True,
            )
    ],
    ),
    
    html.Div([
        dcc.Graph(
            id='Chart'
            ),
        dcc.Interval(
            id='Interval',
            interval=1000,
            n_intervals=0)
    ],
    ),

    html.Br(),
    dcc.Link('Post Processing', href='/page-2'),
])
```
#### Post processing page: 
Support unit transformation through dcc.RadioItems; Overlay an arbitrary number of parameters on a single plot, optionally with
different units and/or data types through dcc.Checklist; Enhance robustness of time-domain and freq-domain analysis through dcc.Slider which allow user to select analysis window.

```
postprocess_layout = html.Div([
    html.H1('Post Processing'),
    html.Div([
        dcc.RadioItems(
            options=[
            {'label':'Radius', 'value':'rad'},
            {'label':'Degree', 'value':'deg'}
            ],
            value='deg',
            id='stripChart_unit',
            inline=True,
            )
    ],
    ),
    
    html.Div([
        dcc.Graph(
            id='stripChart'
            )
    ],
    ),
    
    html.Div([
        html.Label('Time Window'),
        dcc.Slider(
            dataset['time_s'].min(),
            dataset['time_s'].max(),
            id='stripChart_time_slider',
            step=20,
            value=dataset['time_s'].max()-30,
            marks={
            0: {'label': str(min(dataset['time_s']))},
            200: {'label': str(max(dataset['time_s']))}
            },
            )
    ],
    ),
    
    html.Div([
        dcc.Checklist(
            options=[
            {'label':'Roll Accleration Command', 'value':'rollAccelerationCommand_rps2'},
            {'label':'Roll Rate', 'value':'measuredRollRate_rps'},
            {'label':'Roll Command', 'value':'rollAttitudeCommand_rad'},
            {'label':'Roll Attitude', 'value':'measuredRollAttitude_rad'}
            ],
            id='singlePlot_column',
            inline=True,
            )
    ],
    ),

    html.Div([
        dcc.Graph(
            id='time_series'
            )
    ], style={'display': 'inline-block', 'width': '49%'}
    ),
    
    html.Div([
        dcc.Graph(
            id='freq_series'
            )
    ], style={'width': '49%', 'display': 'inline-block', 'padding': '0 20'}
    ),
    
    html.Br(),
    dcc.Link('Start Recording', href='/page-1')
])
```
### Define call back functions




## Authors
Ke-Chu Lee [https://www.linkedin.com/in/ke-chu-lee/]
