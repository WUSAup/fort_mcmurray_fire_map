import pandas as pd
import folium
from folium.plugins import HeatMap
import geopandas as gpd

# ============ 1. 加载和清理 Airbnb 数据 ============
# Airbnb 数据 URL (2024年)
airbnb_url = "http://data.insideairbnb.com/canada/on/toronto/2024-12-01/data/listings.csv.gz"
airbnb_data = pd.read_csv(airbnb_url, compression="gzip")

# 筛选有效字段
airbnb_data = airbnb_data[["id", "latitude", "longitude", "price", "neighbourhood", "room_type"]]

# 清理价格字段并转换为数值
airbnb_data["price"] = airbnb_data["price"].str.replace("$", "").str.replace(",", "").astype(float)

# 检查数据是否涵盖 2024 年 (假设有时间列，如 last_review)
if "last_review" in airbnb_data.columns:
    airbnb_data["last_review"] = pd.to_datetime(airbnb_data["last_review"])
    airbnb_2024 = airbnb_data[airbnb_data["last_review"].dt.year == 2024]
else:
    airbnb_2024 = airbnb_data  # 如果没有年份信息，则默认数据为2024年

# ============ 2. 创建交互式地图 ============
# 创建基础地图
toronto_map = folium.Map(location=[43.7, -79.42], zoom_start=12)

# 分段定义：低价、中价、高价
low_price = airbnb_2024[airbnb_2024["price"] <= 100]
mid_price = airbnb_2024[(airbnb_2024["price"] > 100) & (airbnb_2024["price"] <= 300)]
high_price = airbnb_2024[airbnb_2024["price"] > 300]

# 添加低价房源标记
for _, row in low_price.iterrows():
    folium.CircleMarker(
        location=[row["latitude"], row["longitude"]],
        radius=3,
        color="green",
        fill=True,
        fill_opacity=0.5,
        popup=f"Price: ${row['price']}<br>Neighbourhood: {row['neighbourhood']}"
    ).add_to(toronto_map)

# 添加中价房源标记
for _, row in mid_price.iterrows():
    folium.CircleMarker(
        location=[row["latitude"], row["longitude"]],
        radius=3,
        color="blue",
        fill=True,
        fill_opacity=0.5,
        popup=f"Price: ${row['price']}<br>Neighbourhood: {row['neighbourhood']}"
    ).add_to(toronto_map)

# 添加高价房源标记
for _, row in high_price.iterrows():
    folium.CircleMarker(
        location=[row["latitude"], row["longitude"]],
        radius=3,
        color="red",
        fill=True,
        fill_opacity=0.5,
        popup=f"Price: ${row['price']}<br>Neighbourhood: {row['neighbourhood']}"
    ).add_to(toronto_map)

# ============ 3. 添加热力图图层 ============
low_price_heat = folium.FeatureGroup(name="低价房源热力图")
mid_price_heat = folium.FeatureGroup(name="中价房源热力图")
high_price_heat = folium.FeatureGroup(name="高价房源热力图")

# 添加热力图数据
HeatMap([[row["latitude"], row["longitude"]] for _, row in low_price.iterrows()], radius=10).add_to(low_price_heat)
HeatMap([[row["latitude"], row["longitude"]] for _, row in mid_price.iterrows()], radius=10).add_to(mid_price_heat)
HeatMap([[row["latitude"], row["longitude"]] for _, row in high_price.iterrows()], radius=10).add_to(high_price_heat)

# 将热力图功能组添加到地图
low_price_heat.add_to(toronto_map)
mid_price_heat.add_to(toronto_map)
high_price_heat.add_to(toronto_map)

# 添加图层控制器
folium.LayerControl().add_to(toronto_map)

# ============ 4. 保存地图文件 ============
toronto_map.save("toronto_airbnb_2024_detailed_map.html")
print("地图已保存为 toronto_airbnb_2024_detailed_map.html")
