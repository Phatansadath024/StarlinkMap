# StarlinkMap

import os
import pandas as pd
import sqlalchemy as sa
import folium
from folium.plugins import Search
import geojson

# Database connection setup
engine = sa.create_engine(
    "mssql+pyodbc://LAPTOP-0TPOHUOU\SQLEXPRESS/starlinkdb?"
    "driver=ODBC+Driver+17+for+SQL+Server&"
    "trusted_connection=yes"
)

# SQL query to fetch data including country names
query = '''
SELECT LATITUDE as latitude, LONGITUDE as longitude, LATENCY as latency, UPLOAD_SPEED as upload_speed, 
       DOWNLOAD_SPEED as download_speed, AVAILABLE as available, OTHER_OPTIONS as other_options, 
       LATENCY_1 as latency_1, UPLOAD_SPEED_1 as upload_speed_1, DOWNLOAD_SPEED_1 as download_speed_1, 
       COUNTRY as country
FROM [dbo].[SATARLINK_DB]
'''

# Execute the query and load the data into a DataFrame
sql_query = pd.read_sql_query(query, engine)

# Path to your GeoJSON file
geojson_file = 'C:\\Users\\Sadath\\Downloads\\StarlinkDB\\countries.geojson'

# Check if the GeoJSON file exists
if not os.path.isfile(geojson_file):
    raise FileNotFoundError(f"The GeoJSON file at {geojson_file} was not found.")

# Create a map centered around Europe
map = folium.Map(location=[51.1657, 10.4515], zoom_start=4)  # Centered at Germany for example

# Feature group for the markers to be added to the search functionality
marker_group = folium.FeatureGroup(name="Starlink Locations")

# Function to build the popup HTML
def build_popup_html(column):
    popup_html = '<div style="font-size: 14px;"><b>Starlink Information:</b><br>'
    if column['available'] == 1:
        popup_html += f'<b>Starlink Availability:</b> Yes<br>'
        popup_html += f'<b>Latency:</b> {column["latency"]} ms<br>'
        popup_html += f'<b>Upload Speed:</b> {column["upload_speed"]} Mbps<br>'
        popup_html += f'<b>Download Speed:</b> {column["download_speed"]} Mbps<br>'
    else:
        popup_html += f'<b>Starlink Availability:</b> No<br>'
        if pd.notnull(column['other_options']):
            popup_html += f'<b>Other Options:</b> {column["other_options"]}<br>'
            popup_html += f'<b>Latency 1:</b> {column["latency_1"]} ms<br>'
            popup_html += f'<b>Upload Speed 1:</b> {column["upload_speed_1"]} Mbps<br>'
            popup_html += f'<b>Download Speed 1:</b> {column["download_speed_1"]} Mbps<br>'
    popup_html += '</div>'
    return popup_html

# Loop through the DataFrame and add markers to the map
for index, column in sql_query.iterrows():
    try:
        latitude = float(column['latitude'])
        longitude = float(column['longitude'])
        
        popup_html = build_popup_html(column)
        
        if column['available'] == 1:
            icon_color = 'lightgreen'
        else:
            icon_color = 'red'
        
        folium.Marker(
            location=[latitude, longitude],
            tooltip=folium.Tooltip(f"Country: {column['country']} - Latency: {column['latency']} ms" if pd.notnull(column['latency']) else f"Country: {column['country']} - No latency data"),
            popup=folium.Popup(popup_html, max_width=250),
            icon=folium.Icon(icon="cloud", color=icon_color),
        ).add_to(marker_group)
    except ValueError as e:
        print(f"Error processing column {index}: {e}")
    except KeyError as e:
        print(f"Missing key in column {index}: {e}")

# Add the marker group to the map
marker_group.add_to(map)

# Load the GeoJSON file
with open(geojson_file, 'r') as f:
    geojson_data = geojson.load(f)

# Verify 'ADMIN' field in GeoJSON data
for feature in geojson_data['features']:
    if 'ADMIN' not in feature['properties']:
        raise KeyError(f"The field 'ADMIN' is not available in the properties of the feature: {feature}")

# Style function for GeoJSON layer
def style_function(feature):
    return {
        'fillColor': 'blue',
        'color': 'black',
        'weight': 1.5
    }

# Style function for highlighting search result
def highlight_function(feature):
    return {
        'fillColor': 'white',
        'color': 'yellow',
        'weight': 3
    }

# Create a GeoJSON layer
geojson_layer = folium.GeoJson(
    geojson_data, 
    name='geojson',
    style_function=style_function,
    highlight_function=highlight_function,
    tooltip=folium.GeoJsonTooltip(fields=['ADMIN'])  # Ensure 'ADMIN' is a property in your GeoJSON
).add_to(map)


# Add a search box for GeoJSON data
search_geojson = Search(
    layer=geojson_layer,
    search_label='ADMIN',  # Ensure this matches the property used in the GeoJSON tooltip
    placeholder='Search by country',
    collapsed=False
).add_to(map)

# Save the map to an HTML file
map.save('merged_map.html')

# Display the map
map
