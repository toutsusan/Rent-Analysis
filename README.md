# **基于网络数据与GIS的城市租房“性价比”决策模型**



通过爬取贝壳租房网站的信息获取数据，调用高德地图api计算通勤距离，通过建模获取“性价比得分”作为租房选择的参考，辅助租房决策。



首先先安装必要的python库

    pip install requests 
    pip install beautifulsoup4
    pip install lxml
    pip install pandas
    pip install folium

然后找到自己想爬取的城市，此处以杭州市为例（https://hz.zu.ke.com/zufang）， 务必是此类型页面。
<img width="1895" height="880" alt="image" src="https://github.com/user-attachments/assets/915295c2-c2ea-4314-8b70-b4b958fa50ec" />

确认后复制以下代码到python，按照#号后面的注释修改代码
    
    import requests
    from bs4 import BeautifulSoup
    import pandas as pd
    import time
    import random
    import re
    
    def parse_standard_listing(item):
    
        try:
            title_element = item.find('p', class_='content__list--item--title')
            if not title_element or not title_element.find('a'):
                return None 
    
            detail_link = "贝壳租房网站" + title_element.find('a')['href']  #此处更换自己所查取的网站
            if '/apartment/' in detail_link:
                return None
    
            listing_data = {
                '标题': 'N/A', '价格(元/月)': 0, '区域': 'N/A',
                '户型': 'N/A', '面积(㎡)': 'N/A', '朝向': 'N/A', '详情链接': 'N/A'
            }
            listing_data['标题'] = title_element.get_text(strip=True)
            listing_data['详情链接'] = detail_link
    
            price_element = item.find('span', class_='content__list--item-price')
            if price_element and price_element.find('em'):
                price_text = price_element.find('em').get_text(strip=True)
                listing_data['价格(元/月)'] = int(price_text) if price_text.isdigit() else 0
            else:
                return None 
    
            des_element = item.find('p', class_='content__list--item--des')
            if des_element:
                full_des_text = des_element.get_text(strip=True)
                des_parts = [part.strip() for part in full_des_text.split('/')]
                
                if des_parts:
                    location_info = des_parts.pop(0).replace('·', ' - ')
                    listing_data['区域'] = location_info
    
                for part in des_parts:
                    if '㎡' in part:
                        listing_data['面积(㎡)'] = re.sub(r'\s*㎡', '', part)
                    elif '室' in part or '厅' in part or '卫' in part:
                        listing_data['户型'] = part
                    elif len(part) <= 2 and any(d in part for d in ['东', '南', '西', '北']):
                        listing_data['朝向'] = part
            
            return listing_data
    
        except Exception:
            return None
    
    def get_beike_rent_info_final_v2(max_pages=5):
        base_url = "贝壳租房网站"  #此处更换自己所查取的网站
        headers = {
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36',
        }
        
        all_listings = []
        print(f"--- 自动跳过所有品牌公寓房源 ---")
    
        for page in range(1, max_pages + 1):
            url = f"{base_url}pg{page}/"
            print(f"正在爬取第 {page} 页: {url}")
    
            try:
                response = requests.get(url, headers=headers, timeout=15)
                response.raise_for_status()
                soup = BeautifulSoup(response.text, 'lxml')
                
                listings_on_page = soup.find_all('div', class_='content__list--item')
                if not listings_on_page:
                    print(f"警告: 在第 {page} 页没有找到房源信息，爬取提前结束。")
                    break
    
                count_on_page = 0
                for item in listings_on_page:
                    listing_data = parse_standard_listing(item)
                    if listing_data:
                        all_listings.append(listing_data)
                        count_on_page += 1
                
                print(f"第 {page} 页成功解析 {count_on_page} 条【普通房源】。")
    
                sleep_time = random.uniform(2, 4)
                print(f"休眠 {sleep_time:.2f} 秒...")
                time.sleep(sleep_time)
    
            except requests.exceptions.RequestException as e:
                print(f"请求第 {page} 页时发生网络错误: {e}")
                break
                
        print("--- 爬取结束 ---")
        
        if all_listings:
            return pd.DataFrame(all_listings)
        else:
            return pd.DataFrame()
    
    if __name__ == '__main__':
        PAGES_TO_SCRAPE = 20  #可以把数字改成自己想爬取的页数
    
        rent_data_df = get_beike_rent_info_final_v2(max_pages=PAGES_TO_SCRAPE)
    
        if not rent_data_df.empty:
            try:
                output_filename = '文件保存位置' #选择文件保存路径及命名，保存为csv文件
                columns_order = ['标题', '价格(元/月)', '区域', '户型', '面积(㎡)', '朝向', '详情链接']
                rent_data_df = rent_data_df.reindex(columns=columns_order) 
                
                rent_data_df.to_csv(output_filename, index=False, encoding='utf-8-sig')
                
                print(f"\n成功爬取 {len(rent_data_df)} 条【普通房源】信息。")
                print(f"数据已保存到文件: {output_filename}")
                print("\n数据预览:")
                print(rent_data_df.head())
            except Exception as e:
                print(f"保存文件时出错: {e}")
        else:
            print("\n未能爬取到任何有效的普通房源信息，程序结束。")

确认数据能够爬取且csv文件列为标题、价格(元/月)、区域、户型、面积(㎡)、朝向、详情链接，即可进行下一步

先创建自己的高德地图api，每月的免费额度是够用的，具体申请办法此处不利出，创建时候请选择web服务即可

python另起新一段，输入以下代码，按照#号后面的注释修改代码
    
    import pandas as pd
    import requests
    import time
    import random 
    
    GAODE_API_KEY = "你的api"  #填入高德地图申请的web服务api
      
    COMPANY_ADDRESS = "公司地址"  #填入你公司的地址，从市到区再到具体，例如杭州市西湖区文三路391号
    
    INPUT_CSV = '读取的文件'   #上一步csv文件保存位置
    OUTPUT_CSV = '输出的文件'  #调用api计算后保存的csv文件路径
    
    def get_coordinates(address, api_key):
        geocode_url = "https://restapi.amap.com/v3/geocode/geo"
        params = {'key': api_key, 'address': address, 'city': '你爬取的城市'} #把城市改成之前选定的，如杭州
        try:
            response = requests.get(geocode_url, params=params, timeout=10)
            data = response.json()
            if data['status'] == '1' and data['geocodes']:
                return data['geocodes'][0]['location']
        except Exception as e:
            print(f"  地址解析API请求失败: {e}")
        return None
    
    def get_drive_info(origin, destination, api_key):
        url = "https://restapi.amap.com/v3/direction/driving"
        params = {'key': api_key, 'origin': origin, 'destination': destination}
        try:
            response = requests.get(url, params=params, timeout=10)
            data = response.json()
            if data['status'] == '1' and data.get('route', {}).get('paths'):
                path = data['route']['paths'][0]
                distance_km = round(int(path['distance']) / 1000, 1)
                duration_min = round(int(path['duration']) / 60)
                return f"{distance_km}公里", f"{duration_min}分钟"
        except Exception:
            pass
        return "计算失败", "计算失败"
    
    def get_transit_info(origin, destination, api_key):
        url = "https://restapi.amap.com/v3/direction/transit/integrated"
        params = {
            'key': api_key,
            'origin': origin,
            'destination': destination,
            'city': '你所爬取的城市',  #把城市改成之前选定的，如杭州
            'strategy': '0' 
        }
        try:
            response = requests.get(url, params=params, timeout=10)
            data = response.json()
            if data['status'] == '1' and data.get('route', {}).get('transits'):
                transit = data['route']['transits'][0]
                duration_min = round(int(transit['duration']) / 60)
                return f"{duration_min}分钟"
        except Exception:
            pass
        return "计算失败"
    
    def get_walking_info(origin, destination, api_key):
        url = "https://restapi.amap.com/v3/direction/walking"
        params = {'key': api_key, 'origin': origin, 'destination': destination}
        try:
            response = requests.get(url, params=params, timeout=10)
            data = response.json()
            if data['status'] == '1' and data.get('route', {}).get('paths'):
                path = data['route']['paths'][0]
                distance_km = round(int(path['distance']) / 1000, 1)
                duration_min = round(int(path['duration']) / 60)
                return f"{distance_km}公里", f"{duration_min}分钟"
        except Exception:
            pass
        return "计算失败", "计算失败"
    
    def main():
        if GAODE_API_KEY == "在这里粘贴你从高德开放平台获取的Key":
            print("错误：请先在代码中配置你的高德API Key！")
            return
    
        try:
            df = pd.read_csv(INPUT_CSV)
            print(f"成功读取 {len(df)} 条房源数据自 '{INPUT_CSV}'。")
        except FileNotFoundError:
            print(f"错误：找不到输入文件 '{INPUT_CSV}'。")
            return
    
        print(f"正在获取公司地址 '{COMPANY_ADDRESS}' 的坐标...")
        company_coords = get_coordinates(COMPANY_ADDRESS, GAODE_API_KEY)
        if not company_coords:
            print("无法获取公司坐标，程序终止。")
            return
        print(f"公司坐标获取成功: {company_coords}")
    
        new_columns_data = []
        total = len(df)
        for index, row in df.iterrows():
            rent_address = f"你所爬取的城市{row['区域'].replace(' - ', '')}"  #把城市改成之前选定的，如杭州
            print(f"\n({index + 1}/{total}) 正在处理: {row['标题']}")
            
            commute_info = {
                '驾车距离': '获取坐标失败', '驾车时间': '获取坐标失败',
                '公交时间': '获取坐标失败', '步行距离': '获取坐标失败', '步行时间': '获取坐标失败'
            }
    
            rent_coords = get_coordinates(rent_address, GAODE_API_KEY)
            
            if rent_coords:
                drive_dist, drive_time = get_drive_info(rent_coords, company_coords, GAODE_API_KEY)
                commute_info['驾车距离'] = drive_dist
                commute_info['驾车时间'] = drive_time
                print(f"  驾车: {drive_dist}, {drive_time}")
    
                transit_time = get_transit_info(rent_coords, company_coords, GAODE_API_KEY)
                commute_info['公交时间'] = transit_time
                print(f"  公交: {transit_time}")
    
                walk_dist, walk_time = get_walking_info(rent_coords, company_coords, GAODE_API_KEY)
                commute_info['步行距离'] = walk_dist
                commute_info['步行时间'] = walk_time
                print(f"  步行: {walk_dist}, {walk_time}")
            
            new_columns_data.append(commute_info)
    
            time.sleep(random.uniform(0.1, 0.3))  #控制频率，可以自行更改
    
        commute_df = pd.DataFrame(new_columns_data, index=df.index)
        df_final = pd.concat([df, commute_df], axis=1)
    
        try:
            df_final.to_csv(OUTPUT_CSV, index=False, encoding='utf-8-sig')
            print(f"\n--- 处理完成！---")
            print(f"已将包含多种出行方式的结果保存到新文件: '{OUTPUT_CSV}'")
            print("\n最终数据预览:")
            preview_cols = ['标题', '价格(元/月)', '驾车时间', '公交时间', '步行时间']
            print(df_final[preview_cols].head())
        except Exception as e:
            print(f"保存文件时出错: {e}")
    
    if __name__ == '__main__':
        main()

清洗出来的文件应该是包含标题、价格(元/月)、区域、户型、面积(㎡)、朝向、详情链接、驾车距离、驾车时间、公交时间、步行距离、步行时间

若没问题可进行最后一步，输入以下代码，按照#号后面的注释修改代码
    
    import pandas as pd
    import requests
    import time
    import folium
    from folium.plugins import FeatureGroupSubGroup, MarkerCluster 
    
    GAODE_API_KEY = "你的api"  #此处填入高德地图api
    COMPANY_ADDRESS = "公司地址" #和上一段代码一样的公司地址
    WEIGHTS = {'price': 0.4, 'commute': 0.4, 'area': 0.2}  #此处是修改权重，    'price':价格权重 'commute':通勤权重  'area':面积权重 ，总和一定为1.0
    INPUT_CSV = '读取的数据'  #上一段代码保存的路径
    OUTPUT_CSV_SCORED = '保存的数据'  #建模后保存的csv文件路径
    OUTPUT_MAP_HTML = '保存的地图' #建模地图，尾缀为.html

    
    def get_coordinates_final(area_string, api_key):
    
        parts = area_string.split(' - ')
        keyword = parts[-1] 
        
    
        lat, lon = search_poi_by_keyword(keyword, api_key)
        if lat:
            return lat, lon
    
    
        lat, lon = get_single_coordinate(f"你所爬取的城市{keyword}", api_key) #把城市改成之前选定的，如杭州
        if lat:
            return lat, lon
    
    
        full_address = f"你所爬取的城市{area_string.replace(' - ', '')}" #把城市改成之前选定的，如杭州
        lat, lon = get_single_coordinate(full_address, api_key)
        if not lat:
            print(f"    -> 所有策略均失败: {area_string}")
    
        return lat, lon
    
    def search_poi_by_keyword(keyword, api_key):
        url = "https://restapi.amap.com/v3/place/text"
        params = {
            'key': api_key,
            'keywords': keyword,
            'city': '你所爬取的城市', #把城市改成之前选定的，如杭州
            'citylimit': True, 
            'types': '120302' 
        try:
            response = requests.get(url, params=params, timeout=10)
            data = response.json()
            if data['status'] == '1' and data['pois']:
                location = data['pois'][0]['location'] 
                lon, lat = map(float, location.split(','))
                return lat, lon
        except Exception:
            pass
        return None, None
    
    def get_single_coordinate(address, api_key):
        geocode_url = "https://restapi.amap.com/v3/geocode/geo"
        params = {'key': api_key, 'address': address, 'city': '你所爬取的城市'} #把城市改成之前选定的，如杭州
        try:
            response = requests.get(geocode_url, params=params, timeout=10)
            data = response.json()
            if data['status'] == '1' and data['geocodes']:
                location = data['geocodes'][0]['location']
                lon, lat = map(float, location.split(','))
                return lat, lon
        except Exception:
            pass
        return None, None
    
    def main():
        if GAODE_API_KEY == "在这里粘贴你从高德开放平台获取的Key": #填写你的高德地图api
            print("错误：请先在代码中配置你的高德API Key！")
            return
    
        df = pd.read_csv(INPUT_CSV)
        print("开始数据清洗和特征工程...")
        df['价格(元/月)'] = pd.to_numeric(df['价格(元/月)'], errors='coerce')
        df['面积(㎡)'] = pd.to_numeric(df['面积(㎡)'], errors='coerce')
        df['通勤时间(分钟)'] = pd.to_numeric(df['公交时间'].str.extract(r'([\d\.]+)')[0], errors='coerce')
        df.dropna(subset=['价格(元/月)', '面积(㎡)', '通勤时间(分钟)'], inplace=True)
        print(f"清洗后剩余 {len(df)} 条有效房源。")
        print("开始数据标准化...")
        df['价格得分'] = (df['价格(元/月)'].max() - df['价格(元/月)']) / (df['价格(元/月)'].max() - df['价格(元/月)'].min())
        df['通勤得分'] = (df['通勤时间(分钟)'].max() - df['通勤时间(分钟)']) / (df['通勤时间(分钟)'].max() - df['通勤时间(分钟)'].min())
        df['面积得分'] = (df['面积(㎡)'] - df['面积(㎡)'].min()) / (df['面积(㎡)'].max() - df['面积(㎡)'].min())
        print("计算性价比得分...")
        df['性价比得分'] = ((df['价格得分'] * WEIGHTS['price'] + df['通勤得分'] * WEIGHTS['commute'] + df['面积得分'] * WEIGHTS['area'])* 100).round(1)
    
        print("正在获取房源和公司地理坐标 (已采用最终优化策略)...")
        company_lat, company_lon = get_single_coordinate(COMPANY_ADDRESS, GAODE_API_KEY)
        if not company_lat:
            print("致命错误：无法获取公司坐标，程序终止。")
            return
    
        coords = [get_coordinates_final(row['区域'], GAODE_API_KEY) for index, row in df.iterrows()]
        coord_df = pd.DataFrame(coords, columns=['纬度', '经度'], index=df.index)
        df = pd.concat([df, coord_df], axis=1)
    
        original_count = len(df)
        df.dropna(subset=['纬度', '经度'], inplace=True)
        print(f"成功获取 {len(df)} / {original_count} 条房源的地理坐标。")
        
        df.to_csv(OUTPUT_CSV_SCORED, index=False, encoding='utf-8-sig')
        print(f"\n成功！已将包含性价比得分和坐标的最终数据保存到:\n{OUTPUT_CSV_SCORED}")
    
        print("正在生成交互式地图...")
        m = folium.Map(location=[company_lat, company_lon], zoom_start=12)
        folium.Marker(location=[company_lat, company_lon], popup=f"<strong>公司地址</strong>", icon=folium.Icon(color='red', icon='building', prefix='fa')).add_to(m)
    
    
        marker_cluster = MarkerCluster(name='房源聚合').add_to(m)
        
    
        spider_group = FeatureGroupSubGroup(marker_cluster, '所有房源')
        m.add_child(spider_group)
    
        for idx, row in df.iterrows():
            score = row['性价比得分']
            if score > 80: color = 'green'
            elif score > 60: color = 'blue'
            elif score > 40: color = 'orange'
            else: color = 'gray'
    
            popup_html = f"<b>{row['标题']}</b><hr style='margin: 5px 0;'>性价比得分: <b><font color='{color}'>{score}</font></b><br>价格: {row['价格(元/月)']} 元/月<br>..."
            
    
            folium.CircleMarker(
                location=[row['纬度'], row['经度']],
                radius=5, color=color, fill=True, fill_color=color,
                popup=folium.Popup(popup_html, max_width=300)
            ).add_to(spider_group)
            
        m.save(OUTPUT_MAP_HTML)
        print(f"成功！已将交互式地图保存到:\n{OUTPUT_MAP_HTML}")
    
    
    if __name__ == '__main__':
        main()

