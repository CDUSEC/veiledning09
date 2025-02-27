# 3D-visning av statistikk

Laget av <a href="http://mastermaps.com/">Bjørn Sandvik</a>

I <a href="https://github.com/GeoForum/veiledning08">veiledning 8</a> konstruerte vi et interaktivt kart over befolkningen i Oslo kommune. 
Her skal vi vise de samme dataene i 3D med bruk av WebGL og <a href="http://threejs.org/">Three.js</a>. 

[![3D befolkningskart for Oslo](img/oslo3d.gif)](http://geoforum.github.io/veiledning09/)

Vi bør være forsiktig med bruk av 3D til å visualisere statistikk. Av og til kan det forsvares for å skape oppmerksomhet og interesse, og det kan være lettere å se geografiske variasjoner ved bruk av en ekstra dimensjon. Samtidig vil søylene blokkere for hverandre, og perspektivet gjør det vanskeligere å sammenligne høydene.  

Three.js er et bibliotek som gjør det mye enklere å lage 3D-visualiseringer i nettleseren. Det er ikke laget spesielt for kart, men heldigvis er det lett å konvertere UTM-koordinater til Three.js sitt koordinatsystem. 

### WMS-tjenester fra Kartverket

Vi skal vise søylene oppå det samme bakgrunnskartet som vi brukte i <a href="https://github.com/GeoForum/veiledning08">det 2-dimensjonale kartet</a>, men siden Three.js ikke har støtte for kartfliser (tiles), laster vi inn kartet som ett stort bilde. Vi bruker her <a href="http://kartverket.no/Kart/Gratis-kartdata/WMS-tjenester/">WMS-tjensten til Kartverket</a> for å definere og laste ned kartbildet. Web Map Service (WMS) er en kjent kartstandard som lar deg laste ned kart i ulike projeksjoner og hvor du selv kan bestemme hva som skal vises på kartet (<a href="http://openwms.statkart.no/skwms1/wms.topo2.graatone?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetCapabilities">se oversikt</a>). Det er ikke mulig å laste inn bildet direkte fra Kartverkets server til Three.js, pga. sikkerhetsinnstillingene i nettleseren. I steden lagrer vi en lokal kopi av kartet. 

Vi skal vise befolkningsstatistikk for Oslo kommune, og hvis vi tenker oss en firkant rundt den befolkede delen av kommunen vil denne ha følgende koordinater i UTM 33:

Sørvest: 253700, 6637800 - 
Nordøst: 273800, 6663700

UTM-koordinater er i meter, og dette gir oss et område som er 273 800 - 253 700 = 20 100 meter fra vest til øst, og 6 663 700 - 6 637 800 = 25 900 meter fra nord til sør.  
 
For å hente ut dette kartutsnittet for Oslo kan vi bruke følgende URL: 
 
<a href="http://openwms.statkart.no/skwms1/wms.topo2.graatone?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetMap&CRS=EPSG:32633&BBOX=253700,6637800,273800,6663700&WIDTH=2010&HEIGHT=2590&LAYERS=fjellskygge,N50Vannflate,N50Vannkontur,N50Elver,N50Bilveg,N50Bygningsflate&FORMAT=image/jpeg">http://openwms.statkart.no/skwms1/wms.topo2.graatone?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetMap&CRS=EPSG:32633&BBOX=253700,6637800,273800,6663700&WIDTH=2010&HEIGHT=2590&LAYERS=fjellskygge,N50Vannflate,N50Vannkontur,N50Elver,N50Bilveg,N50Bygningsflate&FORMAT=image/jpeg</a>

Her har vi angitt at kartprojeksjonen skal være UTM 33N (CRS=EPSG:32633), utsnittet er definert av koordinatene over (BBOX=253700,6637800,273800,6663700), oppløsningen skal være 10 meter per pixel (WIDTH=2010, HEIGHT=2590), og vi ønsker å vise følgende kartlag (LAYERS): fjellskygge, vann, elver, bilveg og bygninger. URL'en returnerer dette kartet: 

[![Bakgrunnskart for Oslo](img/wms_oslo.jpg)](https://github.com/GeoForum/veiledning09/blob/gh-pages/data/wms_oslo_topo2_graatone.jpg)

### Bakgrunnskart i Three.js

Vi kan nå sette opp kartet i Three.js. Først definerer vi noen størrelser:

```javascript
var bounds = [253700, 6637800, 273800, 6663700], // UTM 33 vest, sør, øst, nord
    boundsWidth = bounds[2] - bounds[0],
    boundsHeight = bounds[3] - bounds[1],
    sceneWidth = 100,
    sceneHeight = 100 * (boundsHeight / boundsWidth),
    width  = window.innerWidth,
    height = window.innerHeight;
```

"Bounds" er utsnittet i UTM-koordinater som vi omtalt over. "Scene" er bredde og høyde på koordinatsystemet i Three.js. Vi definerer også bredde og høyde på 3D-visningen, som skal dekke hele nettleservinduet. 

```javascript
var scene = new THREE.Scene();

var camera = new THREE.PerspectiveCamera( 20, width / height, 0.1, 1000 );
camera.position.set(0, -200, 120);

var controls = new THREE.TrackballControls(camera);

var renderer = new THREE.WebGLRenderer();
renderer.setSize(width, height);
document.body.appendChild(renderer.domElement);
```

Videre oppretter vi en scene med Three.js hvor vi kan legge til elementer. Vi oppretter også et kamera og angir hvor vi ser på 3D-scenen fra. Vi bruker <a href="http://threejs.org/examples/misc_controls_trackball.html">THREE.TrackballControls</a> som gjør at brukeren selv kan styre kameraet og se på visualiseringen fra ulike vinkler. Til slutt spesifiserer vi at kartet skal tegnes ut med WebGL med bredden og høyden definert over. 

```javascript
var geometry = new THREE.PlaneGeometry(sceneWidth, sceneHeight, 1, 1),
    material = new THREE.MeshBasicMaterial(),
    plane = new THREE.Mesh(geometry, material);

var textureLoader = new THREE.TextureLoader();
textureLoader.load('data/wms_oslo_topo2_graatone.jpg', function(texture) {
    material.map = texture;
    scene.add(plane);
});
```

Selve kartbildet lastes inn ved å først opprette en flate (PlaneGeometry) og legge bildet oppå som en tekstur, før det legges til scenen.

```javascript
function render() {
    controls.update();
    requestAnimationFrame(render);
    renderer.render(scene, camera);
}

render();
```

Vi lager en egen funksjon som tegner ut scenen kontinuerlig ettersom kameravinkelen endrer seg. Vi får nå et <a href="http://geoforum.github.io/veiledning09/kartverket.html">kart i perspektiv</a> som ser slik ut: 

[![Bakgrunnskart for Oslo](img/basemap_3d.png)](http://geoforum.github.io/veiledning09/kartverket.html)

Prøv å endre kameraposisjonen ved å justere verdiene for: camera.position.set(0, -200, 120)

### Statistikk i 3D

Vi er nå klare for å legge på befolkningsdataene fra SSB: 

```javascript
var cellSize = 100,
    xCells = boundsWidth / cellSize,
    yCells = boundsHeight / cellSize,
    boxSize = sceneWidth / xCells,
    valueFactor = 0.02;
    
var colorScale = d3.scale.linear()
    .domain([0, 100, 617])
    .range(['#fec576', '#f99d1c', '#E31A1C']);

var csv = d3.dsv(' ', 'text/plain');

csv('data/Oslo_bef_100m_2015.csv').get(function(error, data) { // ru250m_2015.csv
    for (var i = 0; i < data.length; i++) {
        var id = data[i].rute_100m,
            utmX = parseInt(id.substring(0, 7)) - 2000000 + cellSize, // First seven digits minus false easting
            utmY = parseInt(id.substring(7, 14)) + cellSize, // Last seven digits
            sceneX = (utmX - bounds[0]) / (boundsWidth / sceneWidth) - sceneWidth / 2,
            sceneY = (utmY - bounds[1]) / (boundsHeight / sceneHeight) - sceneHeight / 2,
            value = parseInt(data[i].sum);

        var geometry = new THREE.BoxGeometry(boxSize, boxSize, value * valueFactor);

        var material = new THREE.MeshBasicMaterial({
            color: color(value)
        });

        var cube = new THREE.Mesh(geometry, material);
        cube.position.set(sceneX, sceneY, value * valueFactor / 2);

        scene.add(cube);
    }
});
```

Vi definerere størrelsen på rutene (cellSize) til 100 meter, og regner ut hvor mye det tilsvarer i koordinatsystemet til Three.js (boxSize). Vi lager også en <a href="https://github.com/mbostock/d3/wiki/Quantitative-Scales#linear-scales">lineær fargeskala med D3.js</a>. D3 brukes også til å lese inn dataene, og for hver rute finner UTM-koordinatene fra id'en til ruta (<a href="https://github.com/GeoForum/veiledning08#hvordan-lage-et-rutenett">se detaljer i veiledning 8</a>). UTM-koordinatene blir så konvertert verdier som passer med scenen vi har definert over. 
  
Alle ruter får en søyle eller en boks (BoxGeometry) hvor høyden og fargen bestemmes av antall innbyggere. Søylen plasseres på riktig sted, og legges til scenen vår. 
  
![Befolkningssøyler](img/population_3d.png)
  
Søylene reiser seg nå opp fra kartet, men det er vanskelig å skille søylene fra hverandre. Dette kan vi forbedre ved å legge til et par lyskilder:
  
```javascript  
var dirLight = new THREE.DirectionalLight(0xcccccc, 1);
dirLight.position.set(-70, -50, 80);
scene.add(dirLight);  

var ambLight = new THREE.AmbientLight(0x777777);
scene.add(ambLight);
```

"Directional light" er lys som kommer fra en retning, som fra sola, mens "ambient light" er overalt og siker at det også er litt lys på "baksiden" av søylene. Vi må også endre "materialet" på søylene til et som reflekterer lys (<a href="http://threejs.org/docs/#Reference/Materials/MeshPhongMaterial">MeshPhongMaterial</a>):
  
```javascript  
var material = new THREE.MeshPhongMaterial({
    color: colorScale(value)
});
```  

<a href="http://geoforum.github.io/veiledning09/">Kartet vårt er nå ferdig</a>: 

[![3D befolkningskart for Oslo](img/population_final.png)](http://geoforum.github.io/veiledning09/)

Hvordan synes du visualiserigen ble sammenlignet med <a href="http://geoforum.github.io/veiledning08/">det 2-dimensjonale kartet</a>? 

<hr />
3D display of statistics
Made by Bjørn Sandvik

In guide 8 , we constructed an interactive map of the population in Oslo municipality. Here we will show the same data in 3D using WebGL and Three.js .

3D population map for Oslo

We should be careful about using 3D to visualize statistics. Sometimes it can be defended to create attention and interest, and it can be easier to see geographical variations by using an extra dimension. At the same time, the columns will block each other, and the perspective makes it more difficult to compare the heights.

Three.js is a library that makes it much easier to create 3D visualizations in the browser. It's not made specifically for maps, but luckily it's easy to convert UTM coordinates to Three.js' coordinate system.

WMS services from the Swedish Mapping Authority
We will display the bars on top of the same background map that we used in the 2-dimensional map , but since Three.js does not support map tiles, we will load the map as one large image. Here we use the WMS service of the Mapping Authority to define and download the map image. Web Map Service (WMS) is a well-known map standard that allows you to download maps in various projections and where you can decide for yourself what should be displayed on the map ( see overview ). It is not possible to load the image directly from the Mapping Authority's server to Three.js, because the security settings in the browser. Instead, we store a local copy of the map.

We will show population statistics for the municipality of Oslo, and if we imagine a square around the populated part of the municipality, this will have the following coordinates in UTM 33:

Southwest: 253700, 6637800 - Northeast: 273800, 6663700

UTM coordinates are in meters, and this gives us an area that is 273,800 - 253,700 = 20,100 meters from west to east, and 6,663,700 - 6,637,800 = 25,900 meters from north to south.

To retrieve this map section for Oslo, we can use the following URL:

http://openwms.statkart.no/skwms1/wms.topo2.graatone?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetMap&CRS=EPSG:32633&BBOX=253700,6637800,273800,6663700&WIDTH=2010&HEIGHT=2590&LAYERS=fjellskygge,N50Vannflate,N50Vannkontur,N50Elver,N50Bilveg,N50Bygningsflate&FORMAT=image/jpeg

Here we have specified that the map projection should be UTM 33N (CRS=EPSG:32633), the section is defined by the coordinates above (BBOX=253700,6637800,273800,6663700), the resolution should be 10 meters per pixel (WIDTH=2010, HEIGHT= 2590), and we want to show the following map layers (LAYERS): mountain shadow, water, rivers, road and buildings. The URL returns this map:

Background map for Oslo

Background map in Three.js
We can now set up the map in Three.js. First we define some sizes:

var bounds = [253700, 6637800, 273800, 6663700], // UTM 33 vest, sør, øst, nord
    boundsWidth = bounds[2] - bounds[0],
    boundsHeight = bounds[3] - bounds[1],
    sceneWidth = 100,
    sceneHeight = 100 * (boundsHeight / boundsWidth),
    width  = window.innerWidth,
    height = window.innerHeight;
"Bounds" is the section in UTM coordinates that we mentioned above. "Scene" is the width and height of the coordinate system in Three.js. We also define the width and height of the 3D view, which should cover the entire browser window.

var scene = new THREE.Scene();

var camera = new THREE.PerspectiveCamera( 20, width / height, 0.1, 1000 );
camera.position.set(0, -200, 120);

var controls = new THREE.TrackballControls(camera);

var renderer = new THREE.WebGLRenderer();
renderer.setSize(width, height);
document.body.appendChild(renderer.domElement);
Furthermore, we create a scene with Three.js where we can add elements. We also create a camera and specify where we are viewing the 3D scene from. We use THREE.TrackballControls , which enable the user to control the camera himself and look at the visualization from different angles. Finally, we specify that the map should be drawn with WebGL with the width and height defined above.

var geometry = new THREE.PlaneGeometry(sceneWidth, sceneHeight, 1, 1),
    material = new THREE.MeshBasicMaterial(),
    plane = new THREE.Mesh(geometry, material);

var textureLoader = new THREE.TextureLoader();
textureLoader.load('data/wms_oslo_topo2_graatone.jpg', function(texture) {
    material.map = texture;
    scene.add(plane);
});
The map image itself is loaded by first creating a plane (PlaneGeometry) and placing the image on top as a texture, before adding it to the scene.

function render() {
    controls.update();
    requestAnimationFrame(render);
    renderer.render(scene, camera);
}

render();
We create a separate function that draws out the scene continuously as the camera angle changes. We now get a map in perspective that looks like this:

Background map for Oslo

Try changing the camera position by adjusting the values ​​of: camera.position.set(0, -200, 120)

Statistics in 3D
We are now ready to add the population data from Statistics Norway:

var cellSize = 100,
    xCells = boundsWidth / cellSize,
    yCells = boundsHeight / cellSize,
    boxSize = sceneWidth / xCells,
    valueFactor = 0.02;
    
var colorScale = d3.scale.linear()
    .domain([0, 100, 617])
    .range(['#fec576', '#f99d1c', '#E31A1C']);

var csv = d3.dsv(' ', 'text/plain');

csv('data/Oslo_bef_100m_2015.csv').get(function(error, data) { // ru250m_2015.csv
    for (var i = 0; i < data.length; i++) {
        var id = data[i].rute_100m,
            utmX = parseInt(id.substring(0, 7)) - 2000000 + cellSize, // First seven digits minus false easting
            utmY = parseInt(id.substring(7, 14)) + cellSize, // Last seven digits
            sceneX = (utmX - bounds[0]) / (boundsWidth / sceneWidth) - sceneWidth / 2,
            sceneY = (utmY - bounds[1]) / (boundsHeight / sceneHeight) - sceneHeight / 2,
            value = parseInt(data[i].sum);

        var geometry = new THREE.BoxGeometry(boxSize, boxSize, value * valueFactor);

        var material = new THREE.MeshBasicMaterial({
            color: color(value)
        });

        var cube = new THREE.Mesh(geometry, material);
        cube.position.set(sceneX, sceneY, value * valueFactor / 2);

        scene.add(cube);
    }
});
We define the size of the squares (cellSize) to 100 metres, and calculate how much that corresponds to in the coordinate system of Three.js (boxSize). We also create a linear color scale with D3.js . D3 is also used to read in the data, and for each route the UTM coordinates are found from the ID of the route ( see details in guide 8 ). The UTM coordinates are then converted to values ​​that match the scene we have defined above.

All squares get a column or a box (BoxGeometry) where the height and color are determined by the number of inhabitants. The pillar is placed in the right place, and added to our scene.

Population columns

The pillars now rise up from the map, but it is difficult to separate the pillars from each other. We can improve this by adding a couple of light sources:

var dirLight = new THREE.DirectionalLight(0xcccccc, 1);
dirLight.position.set(-70, -50, 80);
scene.add(dirLight);  

var ambLight = new THREE.AmbientLight(0x777777);
scene.add(ambLight);
"Directional light" is light that comes from one direction, such as from the sun, while "ambient light" is everywhere and means that there is also some light on the "backside" of the pillars. We also need to change the "material" of the columns to one that reflects light ( MeshPhongMaterial ):

var material = new THREE.MeshPhongMaterial({
    color: colorScale(value)
});
Our map is now complete :

3D population map for Oslo

How do you think the visualization compared to the 2-dimensional map ?
