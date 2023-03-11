# 使用D3绘制地图

先看一下效果图:

![svgexport-1](https://s2.loli.net/2023/03/07/wUqtWyEfAnauBST.png)

首先,D3绘制地图的数据[geoJson](https://geojson.org/)可以从[阿里云数据可视化平台](https://datav.aliyun.com/portal/school/atlas/area_selector) 获取:

![image-20230307214327543](https://s2.loli.net/2023/03/07/EW8xKaZdCj2bmfN.png)

然后,地图常用的投影方式中,我选择使用墨卡托投影,这也是百度,高德地图常用的投影方式,

我们需要从数据平台获取一份GeoJson格式的数据,json数据可以直接当成 `Object` 对象来导入使用,也可以使用 `d3.json` 获取远端的Json数据.本次演示采用北京市数据为例

1. 引入D3

   原生直接通过 `<script src="https://d3js.org/d3.v5.js"></script>`

   或者: `npm install d3`

2.为地图绘制创建一个容器:

```html
<body>
    <svg class="map" style="display: inline-block;"> </svg>
</body>
```

3. SVG地图容器的初始化工作:

   ```js
   	const svgW = 1000
       const svgH = 1000
       const colors = ['white', 'blue', 'red'];
       const svg = d3.select(".map")
       svg.attr('viewBox', `0 0 1000 1000`)
       svg.attr("width", svgW).attr("height", svgH)
           .attr('xmlns', 'http://www.w3.org/2000/svg')
   ```

   如果想要实现响应式网页设计,设置 `viewBox` 事半功倍

4. 创建G分组,设定投影参数

   ```js
   const g = svg.append('g').attr("class", 'path-wrap').attr("width", svgW).attr("height", svgH)
   const projection = d3.geoMercator()
           // .fitSize([svgW, svgH], mapJson)
           // 使用北京市的中心经纬度
           .center([116.418757, 39.917544])  //链式写法，.center([longitude, latitude])设置地图中心(真实维度) // 
           .scale([18000])   //.scale([value])设置地图缩放
       // .translate([svgW / 2, svgH / 2]) //.translate([x,y])设置偏移
   ```

   这里要注意一点的是,center需要传入经纬度参数而不是画布参数,这个数据我们可以下载更高一级的地图,比如江苏省的地图,里面会有南京的center经纬度. 使用 `center` `scale` 这种方法属于手动调整地图来合适的展示地图,不过,官方也给了可以自动布局缩放地图适应svg尺寸的工具函数 --  [`fitExtent`](https://github.com/xswei/d3-geo/blob/master/README.md#projection_fitExtent) ,你可以传入一个二位数组定义你画布的左上角和右下角位置

   譬如我们可以这样写:

   ```js
   const projection = d3.geoMercator()
           .fitExtent([[0, 0], [svgW, svgH]], mapJson)
   ```

   或者使用简写:

   ```js
   const projection = d3.geoMercator()
           .fitSize([svgW, svgH], mapJson)
   ```

   `fitSize` 将第一个坐标数据默认设置为 0,0 

   第二个参数需要一个完整的 `geoJson` 对象,注意不是 `geo.features` 这样之后我们就不需要 `center` 与 `scale` 了, 也不能在 `fitExtent `之后调用他们,会报错的.

5. 生成路径数据 绘制地图

   ```js
   const pathGenerator = d3.geoPath()
       .projection(projection)
   ```

   将我们配置好的投影参数传给 `geoPath` 的 `projection` 方法, 可以获取到 `path` 标签的 `d` 属性: [MDN_path](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Element/path)

   然后我们进行数据上的绑定, **`d3.data()` 使用 `geoJson.features` 而非整个 `geoJson`**  一定要注意! 地图颜色方面 就随意使用一个d3 现有的颜色方案填充了,有其他需求可以自行了解配置, 具体教程可以参考: https://github.com/d3/d3-scale-chromatic,

   

   ```js
   const mapDraw = g.selectAll("path")
           .data(mapJson.features) //数据绑定
       mapDraw.join("path")
           .attr("class", "continent-path")
           .attr("d", pathGenerator) //绘制path
           .attr("stroke-width", 1.60347)
           .attr("stroke", "#000")
           .style("fill", (d, i) => d3.schemeSet2[i % 3])
      
   ```

   然后我们可以给我们的地图标注上地理名称,我们配置好的 `projection` 工具可以支持传入一组地理坐标返回一组二维坐标用于Svg元素的定位, 我们可以使用 `geoJson`中的 `centroid` (质心) 作为我们的定位基准,转换之后 传给 `transform` 属性就行了

   ```js
    mapDraw.join('text').attr("class", "continent-text")
           .attr('transform', (d) => {
               return `translate(${projection(d.properties.centroid)})`
           })
           .text((d) => {
               return d.properties.name
           })
   ```

**如果你的地图只有一个小点点,可能你设置错了 `center` 数据, 注意经纬度别传反了**

### 有个坑你可能会踩一下:

**可能第五步你会发现地图无法被渲染出来,可能遇到了下面的问题:**

如果你和我一样从阿里云数据平台下载的地图数据,你可能会和我遇到相同的问题,那就是使用 `center` 与 `scale`  可以正常的显示地图, 使用`fitExtent ` 之后地图就是一个小点或者根本不显示了, 这就是我遇到的第一个坑:

由于D3使用 `ellipsoidal math` 进行地图的投影计算,而不是像其他工具将 `geoJson` 的数据使用笛卡尔坐标系处理, 同时D3 使用右手法则绘制地图,与现有的geoJson 规范相反, 这将导致 绘制区域可能也会被 "取反", 如下图所示:

![屏幕截图 2017-02-19 9 35 16 am](https://s2.loli.net/2023/03/11/15Syv62Gp4zBJUA.png)

![屏幕截图 2017-02-19 9 35 01 am](https://s2.loli.net/2023/03/11/RseogObEn5Ukl2N.png)

我不是相关专业人士,需要更详细解释可以移步 => https://github.com/d3/d3-geo/pull/79#issuecomment-280935242

解决该问题也很简单,我们可以使用 [turf](http://turfjs.org/) 工具 -- 一个可以在浏览器与Node中使用的地理空间数据分析处理工具 来进行转换,

我们需要使用 其中的 `rewind ` 函数 来对我们的数据进行'倒带', 顾名思义 就是磁带中的倒带操作,对geo数据进行反向缠绕操作,\

按需安装自己需要的工具:

`npm i @turf/rewind`

在D3 绘制前处理好 `geoJson` 数据:

```JavaScript
 mapJson.features = mapJson.features.map(function (feature) {
     return turf.rewind(feature, { reverse: true });
 })
```

接下来使用fitExtent来绘制地图了,下面我放一下手动设定参数与使用 `fitExtent` 调整的效果对比

![img](https://s2.loli.net/2023/03/11/aZ9oJTjxBX5l7yu.png)

手动调整缩放与居中问题 确实很难找到合适的大小与偏移值.还是自动调更香啊



下面是完整代码:

```js
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <script src="./resources/map.js"></script>
    <script src="https://d3js.org/d3.v5.js"></script>
    <script src='https://unpkg.com/@turf/turf@6/turf.min.js'></script>
</head>
<style>
    .continent-path {}

    .continent-path:hover {
        fill: aquamarine;
    }
</style>

<body>
    <svg class="map" style="display: inline-block;"> </svg>
</body>

<script>

    const svgW = 1000
    const svgH = 1000
    const colors = ['white', 'blue', 'red'];
    const svg = d3.select(".map")
    svg.attr('viewBox', `0 0 1000 1000`)
    svg.attr("width", svgW).attr("height", svgH)
        .attr('xmlns', 'http://www.w3.org/2000/svg')
        // .attr("stroke-width", 1.60347)
        // .attr("stroke", "#000")

    mapJson.features = mapJson.features.map(function (feature) {
        return turf.rewind(feature, { reverse: true });
    })
    const g = svg.append('g').attr("class", 'path-wrap').attr("width", svgW).attr("height", svgH)
    const projection = d3.geoMercator()
     // .fitExtent([[0, 0], [svgW, svgH]], mapJson) // 使用fitExtent 下面的参数就不要使用了,反之 依然
    // 使用北京市的中心经纬度
    .center([116.418757, 39.917544])  //链式写法，.center([longitude, latitude])设置地图中心(真实维度) 
    .scale([18000])   //.scale([value])设置地图缩放
    .translate([svgW / 2, svgH / 2]) //.translate([x,y])设置偏移

    const pathGenerator = d3.geoPath()
        .projection(projection)
    // console.log('pathGenerator: ', pathGenerator);
    const mapDraw = g.selectAll("path")
        .data(mapJson.features) //数据绑定
    mapDraw.join("path")
        .attr("class", "continent-path")
        .attr("d", pathGenerator) //绘制path
        .attr("stroke-width", 1.60347)
        .attr("stroke", "#000")
        .style("fill", (d, i) => d3.schemeSet2[i % 3])
    mapDraw.join('text').attr("class", "continent-text")
        .attr('transform', (d) => {
            return `translate(${projection(d.properties.centroid)})`
        })
        .text((d) => {
            return d.properties.name
        })

</script>

</html>
```

**全部文件在这里 => https://github.com/3biubiu/D3ToDrawMap**

