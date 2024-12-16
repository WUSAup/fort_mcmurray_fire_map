import pandas as pd
import folium
from folium.plugins import HeatMap
import geopandas as gpd

# ============ 1. Load and Clean Airbnb Data ============
# Airbnb data URL (2024)
airbnb_url = "http://data.insideairbnb.com/canada/on/toronto/2024-12-01/data/listings.csv.gz"
airbnb_data = pd.read_csv(airbnb_url, compression="gzip")

# Filter relevant columns
airbnb_data = airbnb_data[["id", "latitude", "longitude", "price", "neighbourhood", "room_type"]]

# Clean the price field and convert to numeric
airbnb_data["price"] = airbnb_data["price"].str.replace("$", "").str.replace(",", "").astype(float)

# Check if data includes 2024 (assuming a time-related column like `last_review`)
if "last_review" in airbnb_data.columns:
    airbnb_data["last_review"] = pd.to_datetime(airbnb_data["last_review"])
    airbnb_2024 = airbnb_data[airbnb_data["last_review"].dt.year == 2024]
else:
    airbnb_2024 = airbnb_data  # If no year info, assume all data is for 2024

# ============ 2. Create an Interactive Map ============
# Create a base map for Toronto
toronto_map = folium.Map(location=[43.7, -79.42], zoom_start=12)

# Define price ranges: Low, Medium, High
low_price = airbnb_2024[airbnb_2024["price"] <= 100]
mid_price = airbnb_2024[(airbnb_2024["price"] > 100) & (airbnb_2024["price"] <= 300)]
high_price = airbnb_2024[airbnb_2024["price"] > 300]

# Add low-price markers
for _, row in low_price.iterrows():
    folium.CircleMarker(
        location=[row["latitude"], row["longitude"]],
        radius=3,
        color="green",
        fill=True,
        fill_opacity=0.5,
        popup=f"Price: ${row['price']}<br>Neighbourhood: {row['neighbourhood']}"
    ).add_to(toronto_map)

# Add medium-price markers
for _, row in mid_price.iterrows():
    folium.CircleMarker(
        location=[row["latitude"], row["longitude"]],
        radius=3,
        color="blue",
        fill=True,
        fill_opacity=0.5,
        popup=f"Price: ${row['price']}<br>Neighbourhood: {row['neighbourhood']}"
    ).add_to(toronto_map)

# Add high-price markers
for _, row in high_price.iterrows():
    folium.CircleMarker(
        location=[row["latitude"], row["longitude"]],
        radius=3,
        color="red",
        fill=True,
        fill_opacity=0.5,
        popup=f"Price: ${row['price']}<br>Neighbourhood: {row['neighbourhood']}"
    ).add_to(toronto_map)

# ============ 3. Add Heatmap Layers ============
# Create feature groups for different price ranges
low_price_heat = folium.FeatureGroup(name="Low-Price Heatmap")
mid_price_heat = folium.FeatureGroup(name="Mid-Price Heatmap")
high_price_heat = folium.FeatureGroup(name="High-Price Heatmap")

# Add heatmap data
HeatMap([[row["latitude"], row["longitude"]] for _, row in low_price.iterrows()], radius=10).add_to(low_price_heat)
HeatMap([[row["latitude"], row["longitude"]] for _, row in mid_price.iterrows()], radius=10).add_to(mid_price_heat)
HeatMap([[row["latitude"], row["longitude"]] for _, row in high_price.iterrows()], radius=10).add_to(high_price_heat)

# Add heatmap layers to the map
low_price_heat.add_to(toronto_map)
mid_price_heat.add_to(toronto_map)
high_price_heat.add_to(toronto_map)

# ============ 4. Overlay Toronto Boundary ============
# Load Toronto boundary shapefile (replace with your file path)
toronto_boundary = gpd.read_file("path_to_toronto_shapefile.shp")

# Add boundary layer
folium.GeoJson(
    toronto_boundary,
    name="Toronto Boundary",
    style_function=lambda x: {
        "fillColor": "yellow",
        "color": "black",
        "weight": 1,
        "fillOpacity": 0.2
    }
).add_to(toronto_map)

# ============ 5. Add Layer Control ============
folium.LayerControl().add_to(toronto_map)

# Save the map
toronto_map.save("toronto_airbnb_2024_analysis_map.html")
print("Map has been saved as toronto_airbnb_2024_analysis_map.html")

