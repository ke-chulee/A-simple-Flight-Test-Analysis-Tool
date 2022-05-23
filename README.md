# A-simple-Flight-Test-Analysis-Tool

To build a simple Flight-Test Analysis Tool using the Dash-application Python framework that displays the strip charts in real-time, and analyzes recorded signals through interactive-graphing libraries.

## Dependencies
import math
import pandas as pd
import numpy as np
from plotly.subplots import make_subplots
import plotly.graph_objects as go
from scipy.fft import fft, fftfreq, fftshift
from dash import Dash, dcc, html, Input, Output, callback, State

## Executing program
Load data
```
dataset = pd.read_csv("roll_attitude_frequency_sweep.csv")
```

## Authors
Ke-Chu Lee [https://www.linkedin.com/in/ke-chu-lee/]
