# A-simple-Flight-Test-Analysis-Tool

## Dependencies

Import `dash` and `pandas`

Import required module which give us a handful of tools to import and analyze datasets.


```python
# Instaill dash and pandas by 
# `pip install dash` and `pip install pandas`

import math
import pandas as pd
import numpy as np
from plotly.subplots import make_subplots
import plotly.graph_objects as go
from scipy.fft import fft, fftfreq, fftshift
from dash import Dash, dcc, html, Input, Output, callback, State
```

Next, we'll use the `read_csv()` function found in `pandas` to read in a comma-separated values file that contains our data

Note that one should either setting a 'working directory' to simplify importing and exporting: when you ask Python to import (read) a file, or export (write) a file, it will point by default to our chosen working directory, or to type in a full directory path every time.


```python
dataset = pd.read_csv("roll_attitude_frequency_sweep.csv")
```

## Main Program

Dash apps are composed of two parts. The first part is the "layout" of the app and it describes what the application looks like. The second part describes the interactivity of the application.

### App Layout


```python
# Run this app with `python main_combined.py` and
# visit http://127.0.0.1:8050/ in your web browser.

app = Dash(__name__, suppress_callback_exceptions=True)

app.layout = html.Div([
    dcc.Location(id='url', refresh=False),
    html.Div(id='page_content'),
])

if __name__ == '__main__':
    app.run_server(debug=True)
```

Make Dash app using callback functions: functions that are automatically called by Dash whenever an input component's property changes, in order to update some property in another component (the output).


```python
# Home page
home_page = html.Div([
    dcc.Link('Start Recording', href='/page-1'),
])

# Page 1: Display signals at real time
realtime_layout = html.Div([
    # page header
    html.H1('Real Time Data'),
    # input interval and history time window
    html.Div([
        html.Label('Input interval frame [ms]'),
        dcc.Input(
            type='number',
            id='intervalRateMs',
            value='1000',
            min=100, 
            max=200000, 
            step=100,
            ),
        html.Label('Input history time window'),
        dcc.Input(
            type='number',
            id='timeHistoryToDisplay',
            value='30',
            min=0, 
            max=200, 
            step=5,
            )
    ],
    ),

    # set up a radioitem for display unit
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
        # dcc.Graph renders interactive data visualizations
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

# Page 2: Recorded singal analysis
postprocess_layout = html.Div([
    # page header
    html.H1('Post Processing'),
    html.Div([
        # set up a radioitem for display unit
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
        # dcc.Graph renders interactive data visualizations
        dcc.Graph(
            id='stripChart'
            )
    ],
    ),
    
    html.Div([
        html.Label('Time Window'),
        # set up a slider for time window
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
         # set up a markdown of signals to shown
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
    
    # dcc.Graph renders interactive data visualizations
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

#### Define Functions

Define global functions to create time and frequency domain figures. This functions will not be updated/changed.


```python
def create_timeseries(df, index, y_col_name, title):
    fig = go.Figure(go.Scatter(x=dataset.loc[index:len(df),'time_s'],
                          y=np.zeros(len(df)-index),
                          line=dict(color='#000000'),
                          name='baseline'))
    fig.update_xaxes(title_text='Time [s]')
    fig.update_yaxes(title_text='Recorded Signal(s)')
    
    if y_col_name:
        for i in range(0,len(y_col_name),1):
            fig.add_trace(go.Scatter(x=df.loc[index:len(df),'time_s'],
                                     y=df.loc[index:len(df), y_col_name[i]],
                                     name=y_col_name[i]))
        fig.update_layout(showlegend=False)
    
    fig.update_layout(
        margin={'l': 20, 'b': 30, 'r': 10, 't': 10}, 
        hovermode='x unified')
    fig.add_annotation(x=0, y=0.90, 
                       xanchor='left', yanchor='bottom',
                       xref='paper', yref='paper', 
                       showarrow=False, align='left',
                       text=title)
    return fig
```


```python
def create_freqseries(df, index, y_col_name, title):  
    df = df.loc[index:len(df),:]
    N = len(df['time_s'])
    T = 0.01

    fig = go.Figure(go.Scatter(x=np.linspace(-N/2,N/2,N),
                          y=np.zeros(2*N),
                          line=dict(color='#000000'),
                          name='baseline'))
    fig.update_xaxes(title_text='Frequency')
    fig.update_yaxes(title_text='Amplitude')
    
    if y_col_name:
        for i in range(0,len(y_col_name),1):
            y = df.loc[:,y_col_name[i]]
            yf = fft(y.tolist())
            xf = fftfreq(N,T)
            xf = fftshift(xf)
            yplot = fftshift(yf)
            fig.add_trace(go.Scatter(x=xf,
                                     y=1.0/N*np.abs(yplot),
                                     name=y_col_name[i]))
        fig.update_layout(showlegend=False)

    fig.update_layout(
        margin={'l': 20, 'b': 30, 'r': 10, 't': 10}, 
        hovermode='x unified')
    fig.add_annotation(x=0, y=0.90, 
                       xanchor='left', yanchor='bottom',
                       xref='paper', yref='paper', 
                       showarrow=False, align='left',
                       text=title)
    return fig
```

#### Interactivity of The App: Define Call-Back Functions

Use the given/typed URL pathmame to update display pages 


```python
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

Use the input paramaters to update `Interval` state. Disable it when tnext greater than recorded time.


```python
@app.callback(
    Output('Interval', 'disabled'),
    Input('Interval', 'n_intervals'),
    Input('intervalRateMs', 'value'),
    [State('Interval', 'disabled')],
    )
def callback_stop_interval(n_intervals,intervalRateMs,disabled_state):
    tnext = int((n_intervals+1)*int(intervalRateMs)/1000)
    if tnext > 200:
        return not disabled_state
    else:
        return disabled_state
```

Use the input paramaters to update `graph` real-time data figure


```python
@app.callback(
    Output('Chart', 'figure'),
    Input('Interval', 'n_intervals'),
    Input('Chart_unit', 'value'),
    Input('intervalRateMs', 'value'),
    Input('timeHistoryToDisplay', 'value'),
    )
def update_graph(n_intervals,y_unit,intervalRateMs,timeHistoryToDisplay):
    tnow = int(n_intervals*int(intervalRateMs)/1000)
    index = int(np.where(dataset['time_s'].values==tnow)[0])
    if y_unit == 'rad':
        df = dataset
        axisrange = 2,.5,.11
    else:
        df = dataset*180/math.pi
        df['time_s'] = df['time_s']*math.pi/180
        axisrange = 100,20,6
        
    # initialize figures
    fig = make_subplots(
        rows=3, cols=1,
        shared_xaxes=True,
        vertical_spacing=0.05,
        subplot_titles=('Roll Accleration Command','Roll Rate','Roll Attitude')
        )
    # add tracers
    if index <= int(timeHistoryToDisplay):    
        fig.add_trace(go.Scatter(x=df.loc[0:index,'time_s'],
                                y=df.loc[0:index,'rollAccelerationCommand_rps2'],
                                name='pdot cmd'), 
                                row=1, col=1)
        fig.update_yaxes(range = [-axisrange[0],axisrange[0]], row=1, col=1)
        fig.add_trace(go.Scatter(x=df.loc[0:index,'time_s'],
                                y=df.loc[0:index,'measuredRollRate_rps'],
                                name='p'), 
                                row=2, col=1)
        fig.update_yaxes(range = [-axisrange[1],axisrange[1]], row=2, col=1)
        fig.add_trace(go.Scatter(x=df.loc[0:index,'time_s'],
                                y=df.loc[0:index,'rollAttitudeCommand_rad'],
                                name='phi cmd'), 
                                row=3, col=1) 
        fig.add_trace(go.Scatter(x=df.loc[0:index,'time_s'],
                                y=df.loc[0:index,'measuredRollAttitude_rad'],
                                name='phi'), 
                                row=3, col=1) 
        fig.update_yaxes(range = [-axisrange[2],axisrange[2]], row=3, col=1)
    else:
        temp = int(timeHistoryToDisplay)/0.01
        fig.add_trace(go.Scatter(x=df.loc[index-temp:index,'time_s'],
                                y=df.loc[index-temp:index,'rollAccelerationCommand_rps2'],
                                name='pdot cmd'), 
                                row=1, col=1)
        fig.update_yaxes(range = [-axisrange[0],axisrange[0]], row=1, col=1)
        fig.add_trace(go.Scatter(x=df.loc[index-temp:index,'time_s'],
                                y=df.loc[index-temp:index,'measuredRollRate_rps'],
                                name='p'), 
                                row=2, col=1)
        fig.update_yaxes(range = [-axisrange[1],axisrange[1]], row=2, col=1)
        fig.add_trace(go.Scatter(x=df.loc[index-temp:index,'time_s'],
                                y=df.loc[index-temp:index,'rollAttitudeCommand_rad'],
                                name='phi cmd'), 
                                row=3, col=1) 
        fig.add_trace(go.Scatter(x=df.loc[index-temp:index,'time_s'],
                                y=df.loc[index-temp:index,'measuredRollAttitude_rad'],
                                name='phi'), 
                                row=3, col=1) 
        fig.update_yaxes(range = [-axisrange[2],axisrange[2]], row=3, col=1)
        
    # update axis properties
    fig.update_xaxes(title_text='Time [s]', row=3, col=1)
    fig.update_layout(
        margin={'l': 40, 'b': 40, 't': 20, 'r': 0}, 
        showlegend=True)

    return fig
```

Use the input paramaters to update `graph` strip charts figure


```python
@app.callback(
    Output('stripChart', 'figure'),
    Input('stripChart_unit', 'value'),
    Input('stripChart_time_slider', 'value'),
    Input('singlePlot_column', 'value')
    )
def update_strip_graph(y_unit,time_value,y_col_name):
    index = int(np.where(dataset['time_s'].values==time_value)[0])
    if y_unit == 'rad':
        df = dataset
    else:
        df = dataset*180/math.pi
        df['time_s'] = df['time_s']*math.pi/180

    # initialize figures
    fig = make_subplots(
        rows=3, cols=1,
        shared_xaxes=True,
        vertical_spacing=0.05,
        subplot_titles=('Roll Accleration Command','Roll Rate','Roll Attitude')
        )
    # add tracers
    fig.add_trace(go.Scatter(x=df.loc[index:len(df),'time_s'],
                                y=df.loc[index:len(df),'rollAccelerationCommand_rps2'],
                                name='pdot cmd'), 
                                row=1, col=1)
    fig.add_trace(go.Scatter(x=df.loc[index:len(df),'time_s'],
                                y=df.loc[index:len(df),'measuredRollRate_rps'],
                                name='p'), 
                                row=2, col=1)
    fig.add_trace(go.Scatter(x=df.loc[index:len(df),'time_s'],
                                y=df.loc[index:len(df),'rollAttitudeCommand_rad'],
                                name='phi cmd'), 
                                row=3, col=1) 
    fig.add_trace(go.Scatter(x=df.loc[index:len(df),'time_s'],
                                y=df.loc[index:len(df),'measuredRollAttitude_rad'],
                                name='phi'), 
                                row=3, col=1) 
    # update axis properties
    fig.update_xaxes(title_text='Time [s]', row=3, col=1)
    fig.update_layout(
        margin={'l': 40, 'b': 40, 't': 20, 'r': 0}, 
        hovermode='x unified',
        showlegend=True)

    return fig
```

Use the input paramaters to update arguments to defined functions


```python
@app.callback(
    Output('time_series', 'figure'),
    Input('singlePlot_column', 'value'),
    Input('stripChart_unit', 'value'),
    Input('stripChart_time_slider', 'value'),
    )
def update_timeseries(y_column_name, y_unit, time_value):
    index = int(np.where(dataset['time_s'].values==time_value)[0])
    title = '<b>{}</b><br>{}'.format('Time Domain', y_column_name)
    if y_unit == 'rad':
        df = dataset
    else:
        df = dataset*180/math.pi
        df['time_s'] = df['time_s']*math.pi/180

    return create_timeseries(df, index, y_column_name, title)
```


```python
@app.callback(
    Output('freq_series', 'figure'),
    Input('singlePlot_column', 'value'),
    Input('stripChart_unit', 'value'),
    Input('stripChart_time_slider', 'value'),
    )
def update_freqseries(y_column_name, y_unit, time_value):
    index = int(np.where(dataset['time_s'].values==time_value)[0])
    title = '<b>{}</b><br>{}'.format('Frequency Domain', y_column_name)
    if y_unit == 'rad':
        df = dataset
    else:
        df = dataset*180/math.pi
        df['time_s'] = df['time_s']*math.pi/180

    return create_freqseries(df, index, y_column_name, title)
```
