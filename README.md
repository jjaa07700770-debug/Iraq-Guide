
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
body{margin:0;font-family:Arial,sans-serif;background:#f3f6f5}
header{background:#087f5b;color:white;padding:16px;font-size:23px;font-weight:bold}
#status{padding:10px;background:white;font-size:14px}
#map{height:62vh;width:100%}
.buttons{display:grid;grid-template-columns:1fr 1fr;gap:8px;padding:10px}
button{border:0;border-radius:10px;padding:14px 8px;color:white;background:#087f5b;font-size:15px}
.red{background:#c0392b}
.blue{background:#1769aa}
#reports{padding:10px}
.report{background:white;margin:8px 0;padding:12px;border-radius:9px}
</style>
</head>
<body>
<header>🇮🇶 دليل العراق</header>
<div id="status">مرحباً بك في دليل العراق</div>
<div id="map"></div>
<div class="buttons">
<button onclick="locate()">📍 موقعي الحالي</button>
<button class="blue" onclick="destination()">🧭 تحديد وجهة</button>
<button class="red" onclick="report()">🚧 إبلاغ عن خطر</button>
<button onclick="showReports()">📋 عرض البلاغات</button>
</div>
<div id="reports"></div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
const statusBox=document.getElementById('status');
const map=L.map('map').setView([33.3152,44.3661],6);

L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png',{
maxZoom:19,
attribution:'© OpenStreetMap contributors'
}).addTo(map);

let myPosition=null;
let destinationMarker=null;
let routeLine=null;
let reports=[];

function status(t){statusBox.textContent=t;}

function locate(){
 if(!navigator.geolocation){
  status('المتصفح لا يدعم تحديد الموقع');
  return;
 }
 status('جارٍ تحديد موقعك...');
 navigator.geolocation.getCurrentPosition(p=>{
  myPosition=[p.coords.latitude,p.coords.longitude];
  map.setView(myPosition,16);
  L.marker(myPosition).addTo(map).bindPopup('موقعك الحالي').openPopup();
  status('تم تحديد موقعك بنجاح');
 },()=>{
  status('تعذر تحديد الموقع. اسمح للمتصفح باستخدام الموقع GPS');
 },{enableHighAccuracy:true,timeout:20000});
}

function destination(){
 status('اضغط على الخريطة لتحديد وجهتك');
 map.once('click',async e=>{
  const dest=[e.latlng.lat,e.latlng.lng];
  if(destinationMarker)map.removeLayer(destinationMarker);
  destinationMarker=L.marker(dest).addTo(map).bindPopup('الوجهة').openPopup();
  if(!myPosition){
   status('تم تحديد الوجهة. حدد موقعك أولاً لرسم المسار');
   return;
  }
  status('جارٍ البحث عن الطريق...');
  try{
   const url='https://router.project-osrm.org/route/v1/driving/'
    +myPosition[1]+','+myPosition[0]+';'
    +dest[1]+','+dest[0]+'?overview=full&geometries=geojson';
   const res=await fetch(url);
   const data=await res.json();
   if(!data.routes||!data.routes.length)throw Error();
   if(routeLine)map.removeLayer(routeLine);
   routeLine=L.geoJSON(data.routes[0].geometry,{
    style:{color:'#087fcb',weight:6}
   }).addTo(map);
   map.fitBounds(routeLine.getBounds(),{padding:[20,20]});
   const km=(data.routes[0].distance/1000).toFixed(1);
   const mins=Math.round(data.routes[0].duration/60);
   status('المسافة '+km+' كم تقريباً، والوقت '+mins+' دقيقة تقريباً');
  }catch(err){
   status('تعذر حساب الطريق. حاول مرة أخرى');
  }
 });
}

function report(){
 if(!myPosition){
  locate();
  status('حدد موقعك أولاً، ثم اضغط إبلاغ عن خطر');
  return;
 }
 const answer=prompt('اكتب رقم نوع البلاغ:\n1 حادث\n2 حفرة\n3 طريق مغلق\n4 زحام');
 const types={'1':'حادث','2':'حفرة','3':'طريق مغلق','4':'زحام'};
 if(!types[answer])return;
 const item={
  type:types[answer],
  lat:myPosition[0],
  lng:myPosition[1],
  time:new Date().toLocaleString('ar-IQ')
 };
 reports.push(item);
 L.marker([item.lat,item.lng]).addTo(map)
  .bindPopup('بلاغ تجريبي: '+item.type).openPopup();
 status('تم تسجيل البلاغ على هذا الجهاز فقط');
}

function showReports(){
 const box=document.getElementById('reports');
 box.innerHTML='<h3>البلاغات التجريبية</h3>';
 if(!reports.length){
  box.textContent='لا توجد بلاغات في هذه الجلسة';
  return;
 }
 reports.forEach(r=>{
  const div=document.createElement('div');
  div.className='report';
  div.textContent=r.type+' — '+r.time;
  box.appendChild(div);
 });
}
</script>
</body>
</html>
