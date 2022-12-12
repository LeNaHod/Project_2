# 특정 홈페이지를 크롤링하여 좌표정보와 골목상권에대한 좌표를 웹 시각화해보자!

**어느곳을 크롤링해왔는진 오픈x**


### 의도사항:
인터넷 거래사이트에서 판매중인 물품이 오프라인가게가있는지 알고, 가게의 이름과 위치를 저장하고싶다.

### 문제사항:

- 여러번 링크를 타고들어가야한다.
- 가게의 주소가있는곳도있고 없는곳도있다.
- 수동으로 검색하면 가게는 존재하나, 특정사이트에서 접근 할 경로가없는문제
- 가게의 이름과 위치가 기재되어있는 위치가 다르다(div의 위치나, class 네임이다른 경우 등)
- 반경n km위치를 찾기위해 KAKAO API를 이용하려하는데, CURL로 변경해야하는 부분
- 웹사이트 구현을위해 MYSQL에 CSV파일들을 적재해야하는데, 좌표값이 제대로 들어가지않는부분

### 해결사항

- 여러번 링크를 타고 들어가야한다.
-> .click()과 send_keys()로 해결하였다. 대부분은 클릭으로 해결이되지만,선택이 안되고 접근할수없다는 오류(ElementNotVisibleException) 등 여러 오류가 뜬다. 이유는 아직 모르겠다.

- 가게 주소가있고 없는곳이있다.
-> 주소가있는곳만 가져와서 없는 가게리스트를 생성하여 다른 웹사이트에서 검색하는 방법을 채택할수밖에없었다. 

- 수동으로 검색하면 가게가있으나, 접근할 경로가없는문제
-> 이것 또한 위의 문제사항과 같이 다른 웹사이트에서 검색하는 방법 채택
다만 이 문제는 아예 제외를 해야해서  try: except: 를 이용하여 제외하였다.

- 가게의 위치가 기재되어있는 부분이 다른경우
-> 그냥 f12의  XPATH를 카피해오니 오류가 날 수밖에없어서,is_enabled(),is_displayed(),is_selected() 을 이용하여 if 와 섞어 사용해보았으나 해결되지않았다. 해당 클래스의 속성값으로 선택해야하는데 get_attribute(속성이름)은 '속성값'만 가져오는것이라 실패....
그래서 선택한것이 XPATH로, 해당 클래스의 속성값을 이용해서 요소에 접근하여 내용을 가져왔더니 성공. By.XPATH, //*[@속성이름='속성값'/가져올 하위내용] 이런식으로

- 좌표값 적재는 세가지 해결방법이있었는데,
    1.',' 와 공백값 삭제하고 decimal 로 좌표값을 넣어주는방법
    2.POINT 데이터 타입을사용하는방법
    3.MONGODB를 사용하는방법

우리는 시간이 매우 촉박한 관계로 빠른 1번 방법을 선택했다. 하지만 조사에따른결과 좌표값은 될수있으면 POINNT 타입으로 넣어주는게 좋다.(다만, POINT로 넣기위해선 별도의 GEOTEXT값으로 값을 넣어줘야한다는 점)

### ※파이썬 코드로 CURL로 공유되는 API를 성공적으로 바꿨다.

![예시샘플](/python_curl.PNG)

위의 curl rest api를 파이썬 코드로 변환시킨다면 아래와 같은 형식이된다!
```python
#확인해본결과 500m안 카카오프렌즈매장만 반환하는것을 알수있다.
import requests
import json

method='GET'
y='37.514322572335935'
x='127.06283102249932'
radius=20000
params={'query':'카카오프렌즈','y':'37.514322572335935','x':'127.06283102249932','radius':'500'} #기준 m
url='https://dapi.kakao.com/v2/local/search/keyword' #▲ . json은 포맷이므로 requests.get().json으로 잡아줌 ? 뒤의부분은 파라미터에속한다는것을 알수있다.

header={'Authorization': 'KakaoAK 당신의카카오REST서비스키!'}
response=requests.get(url,headers=header,params=params)
tokens=response.json()

print(response)
print(tokens)

```
[카카오개발자사이트](https://developers.kakao.com/docs/latest/ko/local/dev-guide)

[변환참고사이트](https://blog.naver.com/PostView.naver?blogId=blueqnpfr1&logNo=222069986010&parentCategoryNo=&categoryNo=1&viewDate=&isShowPopularPosts=false&from=postView)


### 변경사항

- 웹사이트에 주소정보가 없는 가게들은 검색을 통하여 가져오려했으나, 발생하는 에러 / 경우의 수 가 너무많아서 수동으로 하는것과 다를 바 없어, 오랜 시도끝에 가져오지 않기로함.(다만 코드는 아깝다....)



```python

%pyspark


from selenium import webdriver
from selenium.webdriver.firefox.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver import FirefoxOptions
from selenium.common.exceptions import NoSuchElementException
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from time import sleep #스크롤다운을 위한 대기시간 sleep


#1차크롤링
base_url = "베이스주소입니당"
webdriver_service = Service('/usr/bin/geckodriver')

opts = FirefoxOptions()
opts.add_argument("--headless")		
browser = webdriver.Firefox(service=webdriver_service, options=opts)

#스크롤을내림에 따라 list가증가하니까, 스크롤을 먼저내려줌

SCROLL_PAUASE_TIME=0.8

browser.get(base_url)

while True:
    sleep(SCROLL_PAUASE_TIME)
    #스크롤을 내려준다
    last_height = browser.execute_script("return document.body.scrollHeight")
    browser.execute_script("window.scrollTo(0, document.body.scrollHeight);")
    sleep(SCROLL_PAUASE_TIME)
    new_height = browser.execute_script("return document.body.scrollHeight")
    if new_height == last_height:
        browser.execute_script(
            "window.scrollTo(0, document.body.scrollHeight);")
        sleep(SCROLL_PAUASE_TIME)
        new_height = browser.execute_script(			
            "return document.body.scrollHeight")
        if new_height == last_height:
            break
        else:
            last_height = new_height
            continue

#DATA-TAGET-ID만 추출하여 리스트안에 넣어놓자
classid=browser.find_elements(By.CLASS_NAME,'ui_card--white')

classid_copy=[]
for comment in classid:
    classid_copy.append(comment.get_attribute('data-target-id'))

classid_list=classid_copy

print('1.scesss')
#a태그안의 href(링크)를 추출하고싶다.
# title = browser.find_elements(By.CSS_SELECTOR, 'ui_card__imgcover a')

# for comment in title:
#     print(comment.get_attribute('href'))


#판매작품분기점

final_url=[]
no_url=[]
for class_name in classid_list:
    class_url=f"주소~~~~/{class_name}"
    browser.get(class_url)

    link1=browser.find_element(By.CSS_SELECTOR,'div.artist_card__split a').click()

    try:
        link2=browser.find_element(By.CSS_SELECTOR,'div.ui_grid__item a').get_attribute('href')
        
        final_url.append(link2)
        #print(final_url)
    except NoSuchElementException as ns:
        no_url_name=browser.find_element(By.CLASS_NAME,'artist-info__name').text
        no_url.append(no_url_name)
        #print(no_url)


print(len(final_url))
print(type(final_url))


print(len(no_url))
print(type(no_url))

final_url_dist = list(dict.fromkeys(final_url)) #중복제거

print(len(final_url)) 
print(len(final_url_dist)) 


info_dict={}
info_list=[]
no_info_list=[]


for final_urls in final_url_dist:
    browser.get(final_urls)
    try:
        artist_name=browser.find_element(By.CLASS_NAME,'artist_card__label').text.replace('\n','')
        activetab=browser.find_element(By.XPATH,"//*[@data-ui-id='info-artist']").click()
        store_name=browser.find_element(By.XPATH,"//*[@data-panel-id='info-artist']/tbody/tr[1]/td").text.replace('\n','')
        address=browser.find_element(By.XPATH,"//*[@data-panel-id='info-artist']/tbody/tr[3]/td").text.replace('\n','')
        sellernum=browser.find_element(By.XPATH,"//*[@data-panel-id='info-artist']/tbody/tr[4]/td").text.replace('\n','')# 번호는 굳이 필요없지만, 가져오지 않을 리스트를 만들기위해 일부러 오류
        
        info_list.append([artist_name,store_name,address])
    except NoSuchElementException as e:
        artist_name=browser.find_element(By.CLASS_NAME,'artist_card__label').text.replace('\n','')
        no_info_list.append([artist_name])
        
    
    #print(info_list)
    #print(no_info_list)
    #info_dict['trade_name']=info_list
    #info_dict['adress']=info_list
    #info_dict['artist_name']=info_list

print(type(info_list))
print(info_list)
print(len(no_info_list))
print(info_list)

# 홈페이지에서 크롤링한 정보 베이스 csv파일(추후 x,y,경도,위도 추가)
f = open("idus2.csv", "w",encoding='utf-8')
w = csv.writer(f)
w.writerow(['artist_name','store_name','address','x','y','lot','lat'])
for cont in info_list:
    w.writerow(cont)
f.close()

#위에서 생성한 베이스csv파일을 열어서 주소가없는 셀러 추출하여저장

idus2=spark.read.format('csv').option('header','true').option('inferSchema','true').load('idus2')
idus2.printSchema()
idus2.show()
#idus2.createOrReplaceTempView('idus') 한번생성해도되니까
info_list_notad=spark.sql("""select store_name from idus where address is null""")

info_list_notad.show()

print(type(info_list_notad))

#info_list_notad.write.format('csv').mode('overwrite').save("/na/info_list_notad") #저장이라 한번만

#위의 csv파일을 불러와 리스트로 추출

data = list()
f = open("/home/na/info_no_list.csv",'r')
rea = csv.reader(f)
for row in rea:
    data.append(row)
f.close

#리스트로 반환 후 1차원 리스트로 재반환
no_address_t=data+no_info_list


result_no_url=[]
for no_address_ts in no_address_t:
    result_no_url.extend(no_address_ts)

result_no_url=result_no_url+no_url_dist

print(result_no_url)

#주소가없는셀러+판매링크가없어서 주소가없는셀러 + 예외가 안되었던 셀러
f = open("idus2.csv", "w",encoding='utf-8')
w = csv.writer(f)
w.writerow(['artist_name'])
for cont in result_no_url:
    w.writerow(cont)
    
f.close()

```

### 시도해보다가 의도한대로 안되는코드
```python

div2/div3 등 두가지까지만 처리가능한 코드이다. 하지만 div4가 있는 경우도있기때문에, 이 코드는 제역할을 할수없었다...왜냐면 다 nosuch오류를 반환하기때문에..
    #try:
        #hasp = browser.find_element(By.XPATH,'//*[@id="prd-info"]/p').is_enabled()
    
        # activetab=browser.find_element(By.XPATH,'//*[@id="prd-info"]/div[2]/div[2]/div').click()
        # trade_name=browser.find_element(By.XPATH,'//*[@id="prd-info"]/div[2]/div[2]/table/tbody/tr[1]/td').text 
        # adress=browser.find_element(By.XPATH,'//*[@id="prd-info"]/div[2]/div[2]/table/tbody/tr[3]/td').text


    #except NoSuchElementException as e:
    #        pass
            # activetab=browser.find_element(By.XPATH,'//*[@id="prd-info"]/div[3]/div[2]/div').click()
            # trade_name=browser.find_element(By.XPATH,'//*[@id="prd-info"]/div[3]/div[2]/table/tbody/tr[1]/td').text
            # adress=browser.find_element(By.XPATH,'//*[@id="prd-info"]/div[3]/div[2]/table/tbody/tr[3]/td').text
        
            # info_list1.append(trade_name)
            # info_list2.append(adress)

            # info_dict['trade_name']=info_list1
            # info_dict['adress']=info_list2

            # print(info_list1)
            
 
            # activetab=browser.find_element(By.XPATH,'//*[@id="prd-info"]/div[4]/div[2]/div').click()
            # trade_name=browser.find_element(By.XPATH,'//*[@id="prd-info"]/div[4]/div[2]/table/tbody/tr[1]/td').text
            # adress=browser.find_element(By.XPATH,'//*[@id="prd-info"]/div[4]/div[2]/table/tbody/tr[3]/td').text
        
            # info_list1.append(trade_name)
            # info_list2.append(adress)
        
            # info_dict['trade_name']=info_list1
            # info_dict['adress']=info_list2

            # print(info_list1)

```
## Django

<details>
<summary>MODELS.PY</summary>

```python
(일부)
from django.db import models

class XData(models.Model):
    trdar_nm = models.CharField(db_column='TRDAR_NM', max_length=50, blank=True, null=True)  # Field name made lowercase.
    culture_store_count = models.IntegerField(blank=True, null=True)
    land_price_2020_q1 = models.FloatField(db_column='land_price_2020_Q1', blank=True, null=True)  # Field name made lowercase.
    land_price_2020_q2 = models.FloatField(db_column='land_price_2020_Q2', blank=True, null=True)  # Field name made lowercase.
    land_price_2020_q3 = models.FloatField(db_column='land_price_2020_Q3', blank=True, null=True)  # Field name made lowercase.
    land_price_2020_q4 = models.FloatField(db_column='land_price_2020_Q4', blank=True, null=True)  # Field name made lowercase.
    land_price_2021_q1 = models.FloatField(db_column='land_price_2021_Q1', blank=True, null=True)  # Field name made lowercase.
    land_price_2021_q2 = models.FloatField(db_column='land_price_2021_Q2', blank=True, null=True)  # Field name made lowercase.
    land_price_2021_q3 = models.FloatField(db_column='land_price_2021_Q3', blank=True, null=True)  # Field name made lowercase.
    land_price_2021_q4 = models.FloatField(db_column='land_price_2021_Q4', blank=True, null=True)  # Field name made lowercase.
    market_pop_2020_q1 = models.IntegerField(db_column='market_pop_2020_Q1', blank=True, null=True)  # Field name made lowercase.
    market_pop_2020_q2 = models.IntegerField(db_column='market_pop_2020_Q2', blank=True, null=True)  # Field name made lowercase.
    market_pop_2020_q3 = models.IntegerField(db_column='market_pop_2020_Q3', blank=True, null=True)  # Field name made lowercase.
    market_pop_2020_q4 = models.IntegerField(db_column='market_pop_2020_Q4', blank=True, null=True)  # Field name made lowercase.
    market_pop_2021_q1 = models.IntegerField(db_column='market_pop_2021_Q1', blank=True, null=True)  # Field name made lowercase.
    market_pop_2021_q2 = models.IntegerField(db_column='market_pop_2021_Q2', blank=True, null=True)  # Field name made lowercase.
    market_pop_2021_q3 = models.IntegerField(db_column='market_pop_2021_Q3', blank=True, null=True)  # Field name made lowercase.
    market_pop_2021_q4 = models.IntegerField(db_column='market_pop_2021_Q4', blank=True, null=True)  # Field name made lowercase.
    densitiy = models.FloatField(blank=True, null=True)
    subway_traffic_2020_q1 = models.IntegerField(db_column='subway_traffic_2020_Q1', blank=True, null=True)  # Field name made lowercase.
    subway_traffic_2020_q2 = models.IntegerField(db_column='subway_traffic_2020_Q2', blank=True, null=True)  # Field name made lowercase.
    subway_traffic_2020_q3 = models.IntegerField(db_column='subway_traffic_2020_Q3', blank=True, null=True)  # Field name made lowercase.
    subway_traffic_2020_q4 = models.IntegerField(db_column='subway_traffic_2020_Q4', blank=True, null=True)  # Field name made lowercase.
    subway_traffic_2021_q1 = models.IntegerField(db_column='subway_traffic_2021_Q1', blank=True, null=True)  # Field name made lowercase.
    subway_traffic_2021_q2 = models.IntegerField(db_column='subway_traffic_2021_Q2', blank=True, null=True)  # Field name made lowercase.
    subway_traffic_2021_q3 = models.IntegerField(db_column='subway_traffic_2021_Q3', blank=True, null=True)  # Field name made lowercase.
    subway_traffic_2021_q4 = models.IntegerField(db_column='subway_traffic_2021_Q4', blank=True, null=True)  # Field name made lowercase.
    stdr_ym_cd = models.IntegerField(db_column='STDR_YM_CD', blank=True, null=True)  # Field name made lowercase.
    trdar_se_c = models.CharField(db_column='TRDAR_SE_C', max_length=50, blank=True, null=True)  # Field name made lowercase.
    trdar_se_1 = models.CharField(db_column='TRDAR_SE_1', max_length=50, blank=True, null=True)  # Field name made lowercase.
    trdar_no = models.IntegerField(db_column='TRDAR_NO', blank=True, null=True)  # Field name made lowercase.
    xcnts_valu = models.IntegerField(db_column='XCNTS_VALU', blank=True, null=True)  # Field name made lowercase.
    ydnts_valu = models.IntegerField(db_column='YDNTS_VALU', blank=True, null=True)  # Field name made lowercase.
    signgu_cd = models.IntegerField(db_column='SIGNGU_CD', blank=True, null=True)  # Field name made lowercase.
    adstrd_cd = models.IntegerField(db_column='ADSTRD_CD', blank=True, null=True)  # Field name made lowercase.
    scale = models.FloatField(blank=True, null=True)
    x = models.FloatField(blank=True, null=True)
    y = models.FloatField(blank=True, null=True)
    station_count = models.IntegerField(blank=True, null=True)

    class Meta:
        managed = False
        db_table = 'X_data'


class YData(models.Model):
    trdar_nm = models.CharField(db_column='TRDAR_NM', max_length=50, blank=True, null=True)  # Field name made lowercase.
    ratio_20_q1 = models.FloatField(db_column='ratio_20_Q1', blank=True, null=True)  # Field name made lowercase.
    ratio_20_q2 = models.FloatField(db_column='ratio_20_Q2', blank=True, null=True)  # Field name made lowercase.
    ratio_20_q3 = models.FloatField(db_column='ratio_20_Q3', blank=True, null=True)  # Field name made lowercase.
    ratio_20_q4 = models.FloatField(db_column='ratio_20_Q4', blank=True, null=True)  # Field name made lowercase.
    ratio_21_q1 = models.FloatField(db_column='ratio_21_Q1', blank=True, null=True)  # Field name made lowercase.
    ratio_21_q2 = models.FloatField(db_column='ratio_21_Q2', blank=True, null=True)  # Field name made lowercase.
    ratio_21_q3 = models.FloatField(db_column='ratio_21_Q3', blank=True, null=True)  # Field name made lowercase.
    ratio_21_q4 = models.FloatField(db_column='ratio_21_Q4', blank=True, null=True)  # Field name made lowercase.
    sales_20_q1 = models.IntegerField(db_column='sales_20_Q1', blank=True, null=True)  # Field name made lowercase.
    sales_20_q2 = models.IntegerField(db_column='sales_20_Q2', blank=True, null=True)  # Field name made lowercase.
    sales_20_q3 = models.IntegerField(db_column='sales_20_Q3', blank=True, null=True)  # Field name made lowercase.

```
</details>

<details>
<summary>URLS.PY</summary>

```python

from django.contrib import admin
from django.urls import path,include
from dbtest import views


urlpatterns = [
    path('admin/', admin.site.urls),
    path('',views.home,name='home'),
    path('map/',views.map,name='id_map'),
    path("home_y/",views.home_y,name='home_y'),
    
]
```
</details>

<details>
<summary>views.py</summary>

```python

from django.shortcuts import render,redirect
import folium
from folium.plugins import MarkerCluster
import base64
from folium import IFrame
import folium as g
from .models import *
from folium.plugins import Draw
import pymysql


idus=Idusinfo.objects.all()



def map(request) :
    
    map = g.Map(location=[37.44891, 126.9085],zoom_start=15, width='100%', height='100%',)
    mc = MarkerCluster()
    dro=Draw()
    dro.add_to(map)
    db = pymysql.connect(host='your aws host', port='yourport', user='your dbusername ',
    password='your password', db='your database name', charset='encorder(utf-8..)') 
    cursor1 = db.cursor()
    sql1 = 'select distinct(TRDAR_NO) from polygon_pk' 
    cursor1.execute(sql1)
    # 상권코드 리스트
    store_id_list = cursor1.fetchall() 
    
    for i in store_id_list:
        cursor = db.cursor()
        # 상권 코드 별 좌표(폴리곤 꼭짓점) (DB에 잘못 올려서 y x 순서로 ㅎ;)
        sql = f"select y, x from polygon_pk where TRDAR_NO={i[0]}" 
        cursor.execute(sql)
        result = list(cursor.fetchall())
        # 튜플을 리스트 형태로 변환
        location = [list(result[x]) for x in range(len(result))]
        # print(location)
        # 폴리곤 형식 & 각각 그려넣기
        folium.Polygon(
            locations=location,
            fill=True,
            weight=1,
            tooltip='Polygon',
            color='#3388ff',
            fill_opacity=0.2
        ).add_to(map)
    for i in idus:
       
        marker1=g.Marker([i.lot,i.lat],popup='<a href="https://www.idus.com/c/region/101" target=_blick>아이디어스</a>',tooltip = i.artist_name,
        icon=g.Icon(color='red',icon='bookmark'))
        mc.add_child(marker1)
       
        map.add_child(mc) #그려놓은 경위도의 맵에에다가 해당 경위도에 생성한 마커를찍는다 
    maps=map._repr_html_()
    
    return render(request,'../templates/map.html',{'map' : maps})



#html=============================================================================================================================
def home(request):
    XD=XData.objects.all()
    return render(request,'home.html',{'XD':XD})

def home_y(request):
    YD=YData.objects.all()
    return render(request,'home_y.html',{'YD':YD})

```
</details>



<details>
<summary>templates-html</summary>

```html

map

<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="stylesheet"/>
</head>
<body>
    <!-- 전체 지도 -->
    <div id='map'>
        {{map|safe}}
    </div>
        <title>Data Engineer1_Map</title> 
        <div class="db_1"></div>

</body>


home

<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="stylesheet" href="https://unpkg.com/@picocss/pico@latest/css/pico.min.css">
    <title>Data Engineer1_HOME</title>
    <nav aria-label="breadcrumb">
        <ul>
          <li><a href="{% url 'home' %}">Home</a></li>
          <li><a href="{% url 'id_map' %}">Map</a></li>
          <li><a href="'{% url 'home_y' %}">비율,매출</a></li>
        </ul>
      </nav>
  </head>
  <body>
    <main class="container">
      <h1>서울시 3세대 골목상권 입주를위한 데이터셋 구축</h1>
      <details>
        <summary role="button">상권명</summary>
        <table>
          <thead>
            <tr>
              <th scope="col">상권명</th>
              <th scope="col">밀도</th>
              <th scope="col">면적</th>
              <th scope="col">주변점포갯수</th>
            </tr>
          </thead>
          <tbody>
          {% for xdx in XD %}
            <tr>
              <th scope="row">{{ xdx.trdar_nm }}</th>
              <td>{{xdx.densitiy}}</td>
              <td>{{xdx.scale}}</td>
              <td>{{xdx.culture_store_count}}</td>
          {%endfor%}

          </tbody>
        </table>
        
      </details>
      
      <!-- Secondary -->
      <details>
        <summary role="button" class="secondary">분기별 평당가격(2021)</summary>
        <table>
          <thead>
            <tr>
              <th scope="col">상권명</th>
              <th scope="col">평당가격1분기</th>
              <th scope="col">평당가격2분기</th>
              <th scope="col">평당가격3분기</th>
              <th scope="col">평당가격4분기</th>
            </tr>
          </thead>
          <tbody>
          {% for xdx in XD %}
            <tr>
              <th scope="row">{{xdx.trdar_nm }}</th>
              <td>{{ xdx.land_price_2021_q1 }}</td>
              <td>{{xdx.land_price_2021_q2}}</td>
              <td>{{xdx.land_price_2021_q3}}</td>
              <td>{{xdx.land_price_2021_q4}}</td>
          {%endfor%}

          </tbody>
        </table>
      </details>
      
      <!-- Contrast -->
      <details>
        <summary role="button" class="contrast">분기별 생활인구(2021)</summary>
        <table>
          <thead>
            <tr>
              <th scope="col">상권명</th>
              <th scope="col">분기별 생활인구 1분기</th>
              <th scope="col">분기별 생활인구 2분기</th>
              <th scope="col">분기별 생활인구 3분기</th>
              <th scope="col">분기별 생활인구 4분기</th>
            </tr>
          </thead>
          <tbody>
          {% for xdx in XD %}
            <tr>
              <th scope="row">{{xdx.trdar_nm }}</th>
              <td>{{xdx.market_pop_2021_q1 }}</td>
              <td>{{xdx.market_pop_2021_q2 }}</td>
              <td>{{xdx.market_pop_2021_q3 }}</td>
              <td>{{xdx.market_pop_2021_q4 }}</td>
          {%endfor%}

          </tbody>
        </table>
      </details>

      <details>
        <summary role="button" class="contrast">분기별 지하철승하차량(2021)</summary>
        <table>
          <thead>
            <tr>
              <th scope="col">상권명</th>
              <th scope="col">분기별 지하철승하차량 1분기</th>
              <th scope="col">분기별 지하철승하차량 2분기</th>
              <th scope="col">분기별 지하철승하차량 3분기</th>
              <th scope="col">분기별 지하철승하차량 4분기</th>
              <th scope="col">지하철 개수</th>
            </tr>
          </thead>
          <tbody>
          {% for xdx in XD %}
            <tr>
              <th scope="row">{{xdx.trdar_nm }}</th>
              <td>{{xdx.subway_traffic_2021_q1 }}</td>
              <td>{{xdx.subway_traffic_2021_q2 }}</td>
              <td>{{xdx.subway_traffic_2021_q3 }}</td>
              <td>{{xdx.subway_traffic_2021_q4 }}</td>
              <td>{{xdx.station_count }}</td>
          {%endfor%}

          </tbody>
        </table>
      </details>

    </main>
  </body>
</html>


home_y(y data에관한 정보를 출력하는 html)

<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="stylesheet" href="https://unpkg.com/@picocss/pico@latest/css/pico.min.css">
    <title>Data Engineer1_HOME</title>
    <nav aria-label="breadcrumb">
        <ul>
          <li><a href="{% url 'home' %}">Home</a></li>
          <li><a href="{% url 'id_map' %}">Map</a></li>
          <li><a href="{% url 'home_y' %}">비율,매출</a></li>
        </ul>
      </nav>
  </head>
  <body>
    <main class="container">
      <h1>서울시 3세대 골목상권 입주를위한 데이터셋 구축(비율,매출)</h1>
      <details>
        <summary role="button">분기별 매출 비율(2021)</summary>
        <table>
          <thead>
            <tr>
              <th scope="col">상권명</th>
              <th scope="col">분기별 매출 비율 1분기</th>
              <th scope="col">분기별 매출 비율 2분기</th>
              <th scope="col">분기별 매출 비율 3분기</th>
              <th scope="col">분기별 매출 비율 4분기</th>
            </tr>
          </thead>
          <tbody>
          {% for ydx in YD %}
            <tr>
              <th scope="row">{{ ydx.trdar_nm }}</th>
              <td>{{ydx.ratio_21_q1}}</td>
              <td>{{ydx.ratio_21_q2}}</td>
              <td>{{ydx.ratio_21_q3}}</td>
              <td>{{ydx.ratio_21_q4}}</td>
          {%endfor%}

          </tbody>
        </table>
        
      </details>
      
      <!-- Secondary -->
      <details>
        <summary role="button" class="secondary">분기별 매출(2021)</summary>
        <table>
            <thead>
              <tr>
                <th scope="col">상권명</th>
                <th scope="col">분기별 매출 1분기</th>
                <th scope="col">분기별 매출 2분기</th>
                <th scope="col">분기별 매출 3분기</th>
                <th scope="col">분기별 매출 4분기</th>
              </tr>
            </thead>
            <tbody>
            {% for ydx in YD %}
              <tr>
                <th scope="row">{{ ydx.trdar_nm }}</th>
                <td>{{ydx.sales_21_q1}}</td>
                <td>{{ydx.sales_21_q2}}</td>
                <td>{{ydx.sales_21_q3}}</td>
                <td>{{ydx.sales_21_q4}}</td>
            {%endfor%}
  
            </tbody>
          </table>
      </details>

    </main>
  </body>
</html>
</details>

# 개인 공부

생각보다 유용했던것 

    입력값을 딕셔너리로 구성해주는 코드

    keys = input().split()

    values = map(int, input().split())

    x = dict(zip(keys, values))

    :: {'alpha': 10, 'bravo': 20, 'charlie': 30, 'delta': 40}

csv파일을 불러와서 1차원 리스트로 바꾸기

*csv파일을 리스트로 불러오는 법

    data = list()
    f = open("/home/na/info_no_list.csv",'r')
    rea = csv.reader(f)
    for row in rea:
        data.append(row)
    f.close

＊1차원 리스트로 바꾸는 법

    변수.extend(리스트명) | for 변수 in 리스트 : 변수.extend(리스트)

    리스트+리스트 와 같은 역할을하지만, 만약, a라는 리스트가 2차원, b는 1차원이라 치면 2차원 + 1차원으로 합쳐줌

    하지만, extend는 1차원으로 만들어준다. 

    ※가끔 단어하나하나 분리될때가있으니 주의

    ex)
    no_address_t=data+no_info_list #다차원+다차원


    result_no_url=[]
    for no_address_ts in no_address_t:
        result_no_url.extend(no_address_ts)



    result_no_url=result_no_url+no_url_dist  #no_url_dist는 1차원

**SPARK 열,행 전체보기**
SPARK는 상위20개만 보여준다. 더 많은 행을 보고싶을때는, 아래 명령어를 실행
- df.show(갯수) or df.show(df.count())


**SPARK 저장과 관련하여**

- 변수명or이름.write.format('csv/jdbc/json ...').mode('overwrite / append ..등').save('경로')

-> SPARK 객체(SQL,DF등등)같은 LIST가아닌것. 이름은 배쉬셸에서 변경가능

**하둡 이름변경**
- hdfs dfs -mv /경로/경로..원본파일이름 바꿀경로/바꿀경로..바꿀파일이름. 경로와함께 파일이름도 바꿀수있다.

**그냥 파이썬으로 CSV저장**

LIST일 경우 FOR문으로 넣어줘야한다.

f = open("파일명.csv", "w",encoding='utf-8')
w = csv.writer(f)
w.writerow(['artist_name','store_name','address','lot','lat'])
for cont in info_list:
    w.writerow(cont)
f.close()


웹사이트 배포를위해 mysql에 적재하기위한 DF다듬는 작업

```python

from pyspark.sql.functions import col
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

final_idus=spark.read.option('header','true').option('inferSchema','true').csv('/user/na/Idus.csv')

idus_store_re=final_idus.select(col('artist_name'),col('lot'),col('lat'))
idus_store_re.show()
#idus_store_re.coalesce(1).write.options(header='True', delimiter=',', encoding="utf-8").csv("/na/idus_art")
idus_store_re.coalesce(1).write.options(headr='false',delimiter=',',encoding='utf-8').csv('/na/idus_art_nocol')

웹에서 주소를 보여주진 않을거기때문에 문제가 많이발생하는 베이스 CSV에서 어드레스부분과, 적재시 삭제해줘야하는 컬럼부분을 미리삭제하고
저장하였다.
```

**MYSQL 적재 시 날수있는 오류사항들에대한 정리**

MYSQL에 데이터를 아래와같은 SQL문으로 적재하는과정에서 만난 오류사항들

    #테이블생성문
    create table idusinfo(
    artist_name varchar(300) ,
    lot decimal(15,12),
    lat decimal(16,13)); 
    (플롯은 정확한값(화폐나, 좌표등)을 사용해야할땐 적합하지않음.앞자리6개 소수점5자리까지받는다는의미. decimal(m,d) 인데 m이 d보다 커야한다! )


    #CSV를 MYSQL에 적재하기
    
    로컬에있는 데이터를 적재할경우)
    LOAD DATA LOCAL INFILE '경로' INTO TABLE idusinfo FIELDS TERMINATED BY ',';

    ->*LOCAL*이붙음!

    로컬이 아닌경우)
    LOAD DATA INFILE '경로' INTO TABLE idusinfo FIELDS TERMINATED BY ',';
    (INFILE 뒤에는 CSV파일 경로, TABLE위에는 저장할 테이블(미리만들어둬야한다) BY뒤는 구분자)

    ====오류사항====

    #LOAD DATA에서 나는 --secure-file-priv오류

    - The MySQL server is running with the --secure-file-priv option so it cannot execute this statement

    ->해당오류는 mysql 보안문제상 지정된 경로에있는 파일만 import할수있게 설정되어있기에 나는오류이다.
     
    - 고로, SHOW VARIABLES LIKE "secure_file_priv" 를 입력하여 경로를 확인 후, 그 경로에넣어주던가, 경로를 ""로 널값으로주면,어디에서나 import가 가능하다.

    null값으로 지정시, (sudo사용권장)sudo vim /etc/mysql/my.cnf 으로 들어가서 
    [mysqld]
    secure-file-priv = "" 

    위의내용을 추가해주고 저장하고, 

    - service mysql restart
    ->"서비스"를 재시작해준다.

    #ERROR 3948(42000)에러(로컬LOAD시 발생)
    Loading local data is disabled; this must be enabled on both the client and server sides

    ->위의 보안문제의 연장선이다...  show global variables like 'local_infile'; 을 입력하면 value가 OFF로되어있다. ON으로 바꿔주자!  set global local_infile=true; 아니면 true자리에 1입력도 가능.


    #ERROR 2068(HY000)
    LOAD DATA LOCAL INFILE file request rejected due to restrictions on access. 

    이것도 로컬데이터 로드관련에러이다. 환경변수를 전체적으로 조작해주면되나, 다른문제가 발생할수있으니까 해당부분만 조작해주자

    ->mysql이 설치된 경로로 이동하자(우분투의경우 etc/mysql). my.cnf라는 파일을 찾아 *sudo vim my.cnf* 오픈 안에 아래와같은내용이 있으면 변경, 아니면 추가

    [client]
    local_infile=1

    그리고 저장!



