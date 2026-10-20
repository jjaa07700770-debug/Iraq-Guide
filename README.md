
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="theme-color" content="#087f5b">
<title>دليل العراق</title>

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">

<style>
*{box-sizing:border-box}
body{
 margin:0;
 font-family:Arial,sans-serif;
 background:#f3f6f5;
 color:#222;
}
header{
 background:#087f5b;
 color:white;
 padding:17px;
 font-size:24px;
 font-weight:bold;
 text-align:center;
}
#status{
 padding:12px;
 background:white;
 font-size:14px;
 border-bottom:1px solid #ddd;
}
#searchBox{
 display:flex;
 gap:8px;
 padding:10px;
 background:white;
}
#placeSearch{
 flex:1;
 min-width:0;
 padding:13px 10px;
 border:1px solid #bbb;
 border-radius:9px;
 font-size:16px;
}
button{
 border:0;
 border-radius:9px;
 padding:13px 9px;
 color:white;
 background:#087f5b;
 font-size:15px;
 cursor:pointer;
}
#searchBox button{min-width:65px}
#searchResults{
 background:white;
 padding:0 10px;
}
.search-result{
 display:block;
 width:100%;
 margin-bottom:7px;
 background:#f0f5f3;
 color:#222;
 text-align:right;
 border:1px solid #ddd;
 line-height:1.5;
}
#map{
 height:55vh;
 min-height:320px;
 width:100%;
}
.buttons{
 display:grid;
 grid-template-columns:1fr 1fr;
 gap:8px;
 padding:10px;
}
.blue{background:#1769aa}
.red{background:#c0392b}
#reports{padding:10px}
.report{
 background:white;
 margin:8px 0;
 padding:12px;
 border-radius:9px;
}
footer{
 text-align:center;
 color:#666;
 padding:12px;
 font-size:12px;
}
</style>
</head>

<body>

<header>🇮🇶 دليل العراق</header>

<div id="status">أهلاً بك في دليل العراق</div>

<div id="searchBox">
 <input id="placeSearch"
  type="search"
  placeholder="🔎 ابحث عن مدينة أو شارع أو مكان...">
 <button onclick="searchPlace()">بحث</button>
</div>

<div id="searchResults"></div>

<div id="map"></div>

<div class="buttons">
 <button onclick="locate()">📍 موقعي الحالي</button>
 <button class="blue" onclick="destination()">🧭 تحديد وجهة</button>
 <button class="red" onclick="report()">🚧 إبلاغ عن خطر</button>
 <button onclick="showReports()">📋 عرض البلاغات</button>
 <button class="blue" onclick="clearRoute()">🗺️ مسح المسار</button>
</div>

<div id="reports"></div>

<footer>
 دليل العراق — الخرائط © OpenStreetMap
</footer>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<script>
const statusBox=document.getElementById('status');
const resultsBox=document.getElementById('searchResults');

const map=L.map('map').setView([33.3152,44.3661],6);

L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png',{
 maxZoom:19,
 attribution:'© OpenStreetMap contributors'
}).addTo(map);

let myPosition=null;
let myMarker=null;
let destinationMarker=null;
let routeLine=null;
let destinationPosition=null;
let reportMarkers=[];

let reports=[];
try{
 reports=JSON.parse(localStorage.getItem('iraqGuideReports')||'[]');
 if(!Array.isArray(reports))reports=[];
}catch(e){
 reports=[];
}

function status(message){
 statusBox.textContent=message;
}

function saveReports(){
 try{
  localStorage.setItem('iraqGuideReports',JSON.stringify(reports));
 }catch(e){
  status('تعذر حفظ البلاغات على الجهاز');
 }
}

function locate(){
 if(!navigator.geolocation){
  status('المتصفح لا يدعم تحديد الموقع');
  return;
 }

 status('جارٍ تحديد موقعك عبر GPS...');

 navigator.geolocation.getCurrentPosition(
  p=>{
   myPosition=[
    p.coords.latitude,
    p.coords.longitude
   ];

   if(myMarker)map.removeLayer(myMarker);

   myMarker=L.marker(myPosition)
    .addTo(map)
    .bindPopup('📍 موقعك الحالي');

   map.setView(myPosition,16);
   myMarker.openPopup();

   status('تم تحديد موقعك بنجاح');

   if(destinationPosition){
    routeTo(destinationPosition);
   }
  },
  error=>{
   if(error.code===1){
    status('اسمح للمتصفح بالوصول إلى الموقع GPS');
   }else if(error.code===3){
    status('انتهى وقت تحديد الموقع. حاول مرة أخرى');
   }else{
    status('تعذر تحديد الموقع. تأكد من تشغيل GPS');
   }
  },
  {
   enableHighAccuracy:true,
   timeout:25000,
   maximumAge:0
  }
 );
}

async function searchPlace(){
 const query=document.getElementById('placeSearch').value.trim();

 resultsBox.replaceChildren();

 if(!query){
  status('اكتب اسم المكان الذي تبحث عنه');
  return;
 }

 status('جارٍ البحث عن الأماكن...');

 try{
  const url='https://nominatim.openstreetmap.org/search'
   +'?format=jsonv2'
   +'&q='+encodeURIComponent(query)
   +'&countrycodes=iq'
   +'&limit=5'
   +'&accept-language=ar';

  const response=await fetch(url);

  if(!response.ok)throw Error();

  const places=await response.json();

  if(!places.length){
   status('لم نجد نتائج. جرّب كتابة الاسم بطريقة أخرى');
   return;
  }

  status('اختر المكان الذي تريده');

  places.forEach(place=>{
   const button=document.createElement('button');

   button.className='search-result';
   button.textContent=place.display_name;

   button.onclick=()=>{
    const dest=[
     Number(place.lat),
     Number(place.lon)
    ];

    resultsBox.replaceChildren();
    setDestination(dest,place.display_name);
   };

   resultsBox.appendChild(button);
  });

 }catch(error){
  status('فشل البحث. تحقق من الإنترنت وحاول مجدداً');
 }
}

document.getElementById('placeSearch')
 .addEventListener('keydown',event=>{
  if(event.key==='Enter')searchPlace();
 });

function setDestination(dest,name){
 destinationPosition=dest;

 if(destinationMarker)map.removeLayer(destinationMarker);

 destinationMarker=L.marker(dest)
  .addTo(map)
  .bindPopup(name||'الوجهة')
  .openPopup();

 map.setView(dest,14);

 if(routeLine){
  map.removeLayer(routeLine);
  routeLine=null;
 }

 if(!myPosition){
  status('تم تحديد الوجهة. اضغط موقعي الحالي لحساب الطريق');
  return;
 }

 routeTo(dest);
}

function destination(){
 status('اضغط على الخريطة لتحديد وجهتك');

 map.once('click',event=>{
  const dest=[
   event.latlng.lat,
   event.latlng.lng
  ];

  setDestination(dest,'وجهتك المحددة');
 });
}

async function routeTo(dest){
 if(!myPosition){
  status('حدد موقعك الحالي أولاً');
  return;
 }

 status('جارٍ حساب الطريق...');

 try{
  const url='https://router.project-osrm.org/route/v1/driving/'
   +myPosition[1]+','+myPosition[0]+';'
   +dest[1]+','+dest[0]
   +'?overview=full&geometries=geojson';

  const response=await fetch(url);

  if(!response.ok)throw Error();

  const data=await response.json();

  if(data.code!=='Ok'||!data.routes||!data.routes.length){
   throw Error();
  }

  if(routeLine)map.removeLayer(routeLine);

  routeLine=L.geoJSON(data.routes[0].geometry,{
   style:{
    color:'#087fcb',
    weight:6,
    opacity:0.85
   }
  }).addTo(map);

  map.fitBounds(routeLine.getBounds(),{
   padding:[25,25]
  });

  const km=(data.routes[0].distance/1000).toFixed(1);
  const minutes=Math.round(data.routes[0].duration/60);

  status('المسافة '+km+' كم تقريباً، ووقت القيادة '+minutes+' دقيقة تقريباً');

 }catch(error){
  status('تعذر حساب الطريق. تحقق من الإنترنت أو جرّب وجهة أخرى');
 }
}

function clearRoute(){
 if(routeLine){
  map.removeLayer(routeLine);
  routeLine=null;
 }

 if(destinationMarker){
  map.removeLayer(destinationMarker);
  destinationMarker=null;
 }

 destinationPosition=null;
 status('تم مسح المسار والوجهة');
}

function report(){
 if(!myPosition){
  status('حدد موقعك الحالي أولاً');
  locate();
  return;
 }

 const answer=prompt(
  'اختر نوع البلاغ بكتابة الرقم:\n'
  +'1 حادث\n'
  +'2 حفرة\n'
  +'3 طريق مغلق\n'
  +'4 زحام'
 );

 const types={
  '1':'حادث',
  '2':'حفرة',
  '3':'طريق مغلق',
  '4':'زحام'
 };

 if(!types[answer])return;

 const item={
  type:types[answer],
  lat:myPosition[0],
  lng:myPosition[1],
  time:new Date().toLocaleString('ar-IQ')
 };

 reports.unshift(item);
 saveReports();

 const marker=L.marker([item.lat,item.lng])
  .addTo(map)
  .bindPopup('بلاغ: '+item.type);

 reportMarkers.push(marker);

 status('تم حفظ البلاغ على هذا الجهاز');
}

function showReports(){
 const box=document.getElementById('reports');
 box.replaceChildren();

 const heading=document.createElement('h3');
 heading.textContent='البلاغات المسجلة على هذا الجهاز';
 box.appendChild(heading);

 if(!reports.length){
  box.appendChild(document.createTextNode('لا توجد بلاغات محفوظة'));
  return;
 }

 reports.forEach(item=>{
  const div=document.createElement('div');
  div.className='report';

  const title=document.createElement('strong');
  title.textContent='🚧 '+item.type;

  const details=document.createElement('p');
  details.textContent=item.time;

  const button=document.createElement('button');
  button.className='blue';
  button.textContent='عرض على الخريطة';

  button.onclick=()=>{
   map.setView([item.lat,item.lng],16);
  };

  div.appendChild(title);
  div.appendChild(details);
  div.appendChild(button);

  box.appendChild(div);
 });
}
</script>

</body>
</html>
