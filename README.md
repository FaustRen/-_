# -_測試




好, 其三份程式碼表示三個任務: 爬蟲並簡單處理資料、將資料做集群分析、將分群結果利用python folium套件做地圖的視覺化(html檔), 我將一份一份說明

首先第一份是爬蟲的程式碼, 在這份程式碼裡的saveSheet(res_data, res_data_new, set_new_data_name)是這份程式碼的結束點, 表示我完成爬蟲並準備好資料給下一階段任務,在第一份程式碼裡面的:
def main():
    
    house591_spider = House591Spider()
    ## 篩選條件
    filter_params = filter_params_
    sort_params = sort_params_
    want_page = want_page_
    total_count, houses = house591_spider.search(filter_params, sort_params, want_page=want_page)
    # house_detail = house591_spider.get_house_detail(houses[0]['post_id'])
    print(len(houses))
    print('搜尋結果房屋總數：', total_count)
    
    res_data = house591_spider.getEachHouse(houses)
    # res_data.to_excel('./591data_0906_v7.xlsx')
    ## 轉換 col
    res_data_new = res_data[target_col]
    saveSheet(res_data, res_data_new, set_new_data_name)
    # res_data.to_excel(data_name_to_save)
    df_len = str(len(res_data))
    print(df_len)
#     line_notify("爬完房了Ren",10)
#     line_notify(f"筆數:{df_len}", 1)


if __name__ == "__main__":
    main()

這個部分要請你修改, 看你要改成別的function也好還是怎麼樣, 因為def main跟if __name__ == "__main__":
    main()應該要在所有任務的最下方, 我想一次執行

第一份程式碼完整如下:
import time
import random
import requests
from bs4 import BeautifulSoup
from IPython.display import display
import pandas as pd
# from fake_useragent import UserAgent
import warnings
warnings.filterwarnings('ignore')
from urllib.request import urlopen
##設定參數
want_page_=90
filter_params_ = {'region': '1',
                  'kind': '2',            # 過濾參數(地區,價格,...
                  'rentprice': '7000,15000',
                  'showMore': '1',
                  'area': '7,20',
                  'multiNotice': 'all_sex',}
sort_params_ = {'order': 'money',
                'orderType': 'desc',}
## 爬完Excel檔案設定(名稱,路徑)
set_data_name = "591_0907_1529(new)" # 存擋名稱
data_name_to_save = f"./{set_data_name}.xlsx"
set_new_data_name = "./591_0907_1529(new).xlsx"

## process house

## 
col_dict = {# owner's info
            "linkInfo.tips": "收藏,查看狀態",
            "linkInfo.chargeTxt":"仲介Y/N",
            "linkInfo.mobile":"手機號碼",
            # house info
            "shareInfo.url":"url_link",
            "favData.kindTxt":"類型",
            "favData.price":"價格",
            "favData.address":"地址",
            "favData.area":"坪數",
            "favData.title":"標題",
            "positionRound.lat":"緯度",
            "positionRound.lng":"經度",
            # rule
            "service.rule":"遵守規則",
            "service.desc":"租期與遷入期",
            # compare
            "favData.count":"fav人數",
            "browse.pc": "查看人數(電腦)",
            "browse.mobile":"查看人數(手機)",
            }

target_col=["linkInfo.tips",
            "linkInfo.chargeTxt",
            "linkInfo.mobile",
            
            # house info
            "favData.kindTxt",
            "favData.price",
            "favData.address",
            "favData.area",
            "favData.title",
            "shareInfo.url",
            # rule
            "service.rule",
            "service.desc",
            # compare
            "favData.count",
            "browse.pc",
            "browse.mobile",
            
            # for map
            "positionRound.lng",
            "positionRound.lat",
            "favData.thumb",
            ]






class House591Spider():
    # ua = UserAgent()
    # user_agent = ua.random
    def __init__(self):
        # self.headers = {'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.150 Safari/537.36 Edg/88.0.705.68',}
        # ua = UserAgent()
        # self.user_agent = ua.random
        self.headers = {'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.150 Safari/537.36 Edg/88.0.705.68',}
        # self.headers = {'user-agent': House591Spider.user_agent}
        # self.headers = {'user-agent': self.user_agent}
    
    def getElement(self,url_input,):
        """Record Cookie to get X-CSRF-TOKEN"""
        s = requests.Session()
        url = url_input
        r = s.get(url, headers=self.headers)
        soup = BeautifulSoup(r.text, 'html.parser')
        token_item = soup.select_one('meta[name="csrf-token"]')
        headers = self.headers.copy()
        headers['X-CSRF-TOKEN'] = token_item.get('content')
        return headers,s
        
    def processParams(self,filter_params, sort_params, s):
        """處理參數"""
        params = 'is_format_data=1&is_new_list=1&type=1'
        ## 加上篩選參數，要先轉換為 URL 參數字串格式
        if filter_params:params += ''.join([f'&{key}={value}' for key, value, in filter_params.items()])
        else:params += '&region=1&kind=0'
        ## 在 cookie 設定地區縣市，避免某些條件無法取得資料
        s.cookies.set('urlJumpIp', filter_params.get('region', '1') if filter_params else '1', domain='.591.com.tw')
        ## 排序參數
        if sort_params: params += ''.join([f'&{key}={value}' for key, value, in sort_params.items()])
        return s, params

    def search(self, filter_params=None, sort_params=None, want_page=99):
        """_搜尋房屋 param_"""
        headers,s=self.getElement('https://rent.591.com.tw/')
        url = 'https://rent.591.com.tw/home/search/rsList'          # 搜尋房屋
        s,params = self.processParams(filter_params,sort_params,s)
        total_count = 0
        house_list = []
        page = 0
        
        # get data in each page
        while page < want_page:
            params += f'&firstRow={page*30}'
            r = s.get(url, params=params, headers=headers)
            if r.status_code != requests.codes.ok:
                print('請求失敗', r.status_code)
                break
            page += 1
            data = r.json()
            total_count = data['records']
            house_list.extend(data['data']['data'])
            # 隨機 delay 一段時間
            # time.sleep(random.uniform(2, 3))

        return total_count, house_list

    def get_house_detail(self, house_id):
        """房屋詳情"""
        # 紀錄 Cookie 取得 X-CSRF-TOKEN, deviceid
        s = requests.Session()
        url = f'https://rent.591.com.tw/home/{house_id}'
        r = s.get(url, headers=self.headers)
        soup = BeautifulSoup(r.text, 'html.parser')
        token_item = soup.select_one('meta[name="csrf-token"]')
        
        ## headers
        headers = self.headers.copy()
        headers['X-CSRF-TOKEN'] = token_item.get('content')
        
        if s.cookies.get_dict()['T591_TOKEN']:
            headers['deviceid'] = s.cookies.get_dict()['T591_TOKEN']
            # print(headers['deviceid'])
        headers['device'] = 'pc'
        # headers['token'] = s.cookies.get_dict()['PHPSESSID']
        
        url = f'https://bff.591.com.tw/v1/house/rent/detail?id={house_id}'
        r = s.get(url, headers=headers)
        if r.status_code != requests.codes.ok:
            print('請求失敗', r.status_code)
            return
        
        ## house data
        house_detail = r.json()['data']
        return house_detail
    
    def getEachHouse(self,house_data_input,):
        data_0 = self.get_house_detail(house_data_input[0]['post_id'])
        df0 = (pd.DataFrame(pd.json_normalize(data_0)))
        
        try:
            for i in range(1, len(house_data_input)):
                each_house_detail = self.get_house_detail(house_data_input[i]['post_id'])
                each_house = (pd.DataFrame(pd.json_normalize(each_house_detail)))
                df0 = df0.append(each_house)
            return df0
        except:
            return df0
    
def line_notify(message,notify_time):
    LINE_URL = 'https://notify-api.line.me/api/notify'
    LINE_TOKEN = "自行設定Token"
    
    for _ in range(notify_time):
        requests.post(
            LINE_URL,
            headers={'Authorization': f'Bearer {LINE_TOKEN}'},
            data={'message': message})

def saveSheet(data1_input,data2_input,path_input=set_data_name):
    writer = pd.ExcelWriter(path=path_input, engine='xlsxwriter')
    data1_input.to_excel(writer, sheet_name = 'raw_data')
    data2_input.to_excel(writer, sheet_name = 'target_data')
    writer.save()
    writer.close()



def main():
    
    house591_spider = House591Spider()
    ## 篩選條件
    filter_params = filter_params_
    sort_params = sort_params_
    want_page = want_page_
    total_count, houses = house591_spider.search(filter_params, sort_params, want_page=want_page)
    # house_detail = house591_spider.get_house_detail(houses[0]['post_id'])
    print(len(houses))
    print('搜尋結果房屋總數：', total_count)
    
    res_data = house591_spider.getEachHouse(houses)
    # res_data.to_excel('./591data_0906_v7.xlsx')
    ## 轉換 col
    res_data_new = res_data[target_col]
    saveSheet(res_data, res_data_new, set_new_data_name)
    # res_data.to_excel(data_name_to_save)
    df_len = str(len(res_data))
    print(df_len)
#     line_notify("爬完房了Ren",10)
#     line_notify(f"筆數:{df_len}", 1)


if __name__ == "__main__":
    main()


接下來是第二份程式碼, 在上述的第一階段程式碼裡, 整合後會得到一份資料, 第二份程式碼則利用那份資料進行集群分析, 程式碼如下:

import os
import warnings
import pandas as pd
import matplotlib as mpl
from matplotlib import pyplot as plt
from IPython.display import display
from sklearn.cluster import KMeans


warnings.filterwarnings('ignore')
mpl.rcParams.update(mpl.rcParamsDefault)
plt.rcParams['font.sans-serif'] = ['Arial Unicode MS']
plt.rcParams['axes.unicode_minus'] = False # 顯示負號

from sklearn.preprocessing import MinMaxScaler
from sklearn.metrics import silhouette_score
from sklearn.metrics import calinski_harabasz_score
from sklearn.metrics import davies_bouldin_score
scale = MinMaxScaler()


df = pd.read_excel("./591_0907_1529(new).xlsx",sheet_name="target_data")
df['total_browse'] = df['browse.mobile']+df['browse.pc']
print(len(df))

#%%


def normalized_func(df_input):
    """ 正規化 """
    
    ## 保留index, Columns'name, 處理時會將此弄不見
    copy_idx = df_input.index
    copy_col = list(df_input.columns)
    normalized_df = df_input[copy_col].copy()
    
    ## scale
    normalized_df = pd.DataFrame(scale.fit_transform(normalized_df))
    
    ## 還原 index, 欄位名稱
    normalized_df = normalized_df.set_index(copy_idx)
    normalized_df.columns = copy_col
    return normalized_df

def kmeans_func(cluster_number, normalized_df):
    """ K-Means Model"""
    # kmeans = KMeans(n_clusters=cluster_number, random_state=0, init='k-means++',max_iter=300,n_init=60)
    
    kmeans = KMeans(n_clusters=cluster_number, random_state=0, init='k-means++',max_iter=500,n_init=10,tol=0.0001)
    kmeans.fit(normalized_df)
    
    Custcenter = kmeans.cluster_centers_
    Custcenter_reverse = scale.inverse_transform(Custcenter)
    labels = kmeans.labels_
    
    ## 
    cluster_df = normalized_df.copy(deep=True)
    cluster_df['Cluster'] = labels
    # correct = correct*1
    # correct_labels = sum(y== labels)
    
    sulhoutte_score = silhouette_score(normalized_df,labels)
    DB_score = davies_bouldin_score(normalized_df, labels)
    
    ## print score
    print("DB 指數",DB_score)
    print("CH 指數",calinski_harabasz_score(normalized_df,labels))
    
    inertia = kmeans.inertia_
    return kmeans, Custcenter, Custcenter_reverse, cluster_df, sulhoutte_score, inertia, labels

def clusteringHouse():
    data_for_clustering = df[['favData.price','favData.area','favData.count']].copy()
    clustering_col = list(data_for_clustering.columns)
    cluster_number=6
    clustering_data1 = data_for_clustering.copy()
    normalized_df1=clustering_data1.copy()
    normalized_df1=normalized_func(normalized_df1[clustering_col]) # selcet clustering data

    clustering_data_input = normalized_df1.copy() # original rfm
    copy_features=data_for_clustering.copy()
    ## excute
    kmeans, Custcenter, Custcenter_reverse, cluster_df, score, inertia, labels =  kmeans_func(cluster_number, clustering_data_input)

## review_data


    ## review
    print("-----"*10)
    print(f"群數: {cluster_number}")
    print(f"WCSS Intertia: {inertia}")
    print(f"輪廓係數: {score}")
    print(f"套房數: {len(clustering_data_input)}")
    print("-----"*10)

    ## 看各群筆數
    display(cluster_df.value_counts("Cluster").to_frame())
    copy_features=data_for_clustering.copy()
    print(f"筆數 {len(copy_features)}")
    copy_features['Cluster'] = cluster_df['Cluster']
    copy_features.to_csv("./Cluster_data_0907_v3.csv")
    print("ok")

## 執行分群
copy_features = clusteringHouse()

以上的第二份程式碼當中, copy_features = clusteringHouse()為執行點

接下來是第三份程式碼, 這份程式碼利用folium套件將前面階段所準備的資料進行視覺化,
最後儲存成html檔, 檔案名稱為maptest_0118.html, 完整程式碼如下:
#%%
import folium
import pandas as pd
from datetime import datetime as dt
today = str(dt.now())[:10]
now = str(dt.now())[11:16]



## 準備資料
df = pd.read_excel("./591_0907_1529(new).xlsx",sheet_name="target_data")# target data(有分群)
df_raw=pd.read_excel("./591_0907_1529(new).xlsx",sheet_name="raw_data") # raw data
cluster_data = pd.read_csv("./Cluster_data_0907_v3.csv")                # 分群資料
## 建立欄位
df['total_browse'] = df['browse.mobile']+df['browse.pc']                # 總瀏覽人數
df['Cluster']=cluster_data['Cluster']                                   # 分群編號
## 建立房屋地圖時用的
cluster_label_list = list(df.value_counts('Cluster').to_frame().index)
cluster_label_list





def cal設備數量(string_input):
    """計算每一租屋設備符合數量"""
    cal_num = 0
    for i in range(15): cal_num+=eval(string_input)[i]['active']
    return cal_num

def createDeviceList(device_list, df_raw):
    """get each room's device's quantity"""
    for each_str in range(len(df_raw)):
        each_str = df_raw["service.facility"][each_str]
        each_num = cal設備數量(each_str)
        device_list.append(each_num)
    return device_list
    
def prepareData(df, df_raw, cluster_data):
    """Process the data"""
    device_list=createDeviceList([],df_raw)
    df['total_browse'] = df['browse.mobile']+df['browse.pc']                # 總瀏覽人數
    df['Cluster']=cluster_data['Cluster']                                   # 分群編號
    df['設備數量']=device_list                                               # 設備數量
    df['publish_status'] = df_raw['publish.name'].fillna('-')
    return df



## 準備資料
df = prepareData(df, df_raw, cluster_data)

# 根據條件給予顏色
fav_score_50 = (df['favData.count'].describe()['50%'])
fav_score_75 = (df['favData.count'].describe()['75%'])
browse_score_50 = df['total_browse'].describe()['50%']
browse_score_75 = df['total_browse'].describe()['75%']


# input data to html
def folium_html(house_name, 
                fav_count, 
                house_link, 
                house_address,
                price_input,
                house_area,
                house_rule,
                device_num_input,
                publish_status_input,
                ):
    
    hot_status_input = f"{str(publish_status_input)}"
    rent_price = f"租金: {str(price_input)}"
    room_space = f"坪數: {str(house_area)}"
    room_rule = str(house_rule)    
    Reviewer_Number = f'瀏覽人數: {str(browse_num)}'
    fav_number = f'收藏人數: {str(fav_count)}'
    each_device_num =f'設備數量： {str(device_num_input)}/15'
    google_link = 'https://www.google.com/search?q='+house_address

    ## create html
    pop_html = folium.Html(f"""<p style="text-align: center;"><span style="font-family: Didot, serif; font-size: 21px;">{house_name}</span></p>
    <p style="text-align: center;"><span style="font-family: Didot, serif; font-size: 14px;">{hot_status_input}</span></p>
    <p style="text-align: center;"><iframe src={img_src}embed width="250" height="170" frameborder="0" scrolling="auto" allowtransparency="true"></iframe>
    <p style="text-align: center;"><span style="font-family: Didot, serif; font-size: 14px;">{rent_price}</span></p>
    <p style="text-align: center;"><span style="font-family: Didot, serif; font-size: 14px;">{room_space}</span></p>
    <p style="text-align: center;"><span style="font-family: Didot, serif; font-size: 14px;">{room_rule}</span></p>
    <p style="text-align: center;"><span style="font-family: Didot, serif; font-size: 14px;">{each_device_num}</span></p>
    <p style="text-align: center;"><span style="font-family: Didot, serif; font-size: 14px;">{fav_number}</span></p>
    <p style="text-align: center;"><a href={house_link} target="_blank" title="Let's 591 to {house_name}"><span style="font-family: Didot, serif; font-size: 15px;">Let's 591 to {house_name}</span></a></p>
    <p style="text-align: center;"><a href={google_link} target="_blank" title="Google {house_address}"><span style="font-family: Didot, serif; font-size: 15px;">Google {house_address}</span></a></p>
    """, script=True)
    return pop_html



def House_color_icon(cluster_label_input):
    if cluster_label_input==cluster_label_list[5]:
        return 'red', 'glyphicon glyphicon-star', 'glyphicon'  # > 75    
    elif cluster_label_input==cluster_label_list[4]:
        return 'orange', 'fa-adjust', 'fa'  # <25
    elif cluster_label_input==cluster_label_list[3]:
        return 'white', 'fa-adjust', 'fa'  # <25
    elif cluster_label_input==cluster_label_list[2]:
        return 'blue', 'fa-adjust', 'fa'
    else:
        return 'green', 'fa-adjust', 'fa'



foodpand_map = folium.Map(location=[25.00, 121.53], zoom_start=12,crs='EPSG3857')
# start
for i in range(len(df)):
    lon = df['positionRound.lng'][i]
    if int(lon)<100:
        continue
    lat = df['positionRound.lat'][i]
    price = df['favData.price'][i]
    if int(price)>13000:
        continue
    picture_link = df['favData.thumb'][i]
    house_link = df['shareInfo.url'][i]
    house_title = df['favData.title'][i]
    house_address = df['favData.address'][i]

    fav_count = df['favData.count'][i]
    img_src = picture_link
    browse_num = df['total_browse'][i]
    house_rule = df['service.rule'][i]
    house_area = df['favData.area'][i]
    each_cluster = df['Cluster'][i]
    device_num = df['設備數量'][i]
    each_publish_status_input = df['publish_status'][i]

    # pop_html = folium_html(restaurant_name, Restaurant_rate, pop_web, reviewer_number)
    pop_html = folium_html(house_title, fav_count, house_link, house_address,price,house_area,house_rule,device_num,each_publish_status_input)
    plot_color, plot_icon, plot_prefix = House_color_icon(each_cluster)
    # 顯示資訊
    # pop_text = restaurant_name+':'+Restaurant_rate
    pop_text = house_rule
    popup = folium.Popup(pop_html, max_width=220)

    foodpand_map.add_child(folium.Marker(location=[lat, lon],
                                         icon=folium.Icon(icon=plot_icon, color=plot_color, prefix=plot_prefix),
                                         popup=popup,parse_html=True ))
foodpand_map


foodpand_map.save('maptest_0118.html')
search_text = "cdn.jsdelivr.net"
replace_text = "gcore.jsdelivr.net"
with open(r'maptest_0118.html','r' ,encoding='UTF-8') as file:
    data = file.read()
    data = data.replace(search_text, replace_text)

with open(r'maptest_0118.html', 'w', encoding='UTF-8') as file:
    file.write(data)
# %%

以上所有程式碼當中, 除了整合以外我希望你做到幾件事, 首先將import 套件的部分全部移到最上方, 然後是將三份程式碼分成三個階段, 最後用if __name__ == "__main__":
    main()
讓我一次執行, 因此在第一份程式碼的if __name__ == "__main__":
    main()
這個部分要請你做修改, 麻煩你試試看了




