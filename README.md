# webGL_practice
This practice is based on Homework 1 of ICG@NTU. 

## Goal
* Flat, Gouraud, and Phong shading with Phong reflection model in shaders.
* Multiple transformations (four fundamental transforms) on objects in a scene.
* At least 3 objects & at least 3 light sources


## Code Notes

```
function initShaders() {
    var fragmentShader = getShader(gl, "fragmentShader"); //初始化摻數
    var vertexShader   = getShader(gl, "vertexShader");

    shaderProgram = gl.createProgram();  //每個program 都會綁定一個fragmentShader/vertexShader，不同的shading 就會需要有不同的program
    gl.attachShader(shaderProgram, vertexShader);
    gl.attachShader(shaderProgram, fragmentShader);
    gl.linkProgram(shaderProgram);



    shaderProgram.pMatrixUniform  = gl.getUniformLocation(shaderProgram, "uPMatrix");   
    // 做transform matrix的時候需要用到


    // Setup Projection Matrix  
    mat4.perspective(45, gl.viewportWidth / gl.viewportHeight, 0.1, 100.0, pMatrix); // FOV=45 degree

    // Setup Model-View Matrix
    mat4.identity(mvMatrix); // 把 my Matrix 建成單位矩陣
    
    mat4.translate(mvMatrix, [0, 0, -40]); //先做 translate。 -40 是因為 webGL 預設將 camera 放在 (0,0,0) 看向 (0,0,-1)，所以東西都要擺在 負z才看得到。離原點越近，東西就會越大。 
    mat4.rotate(mvMatrix, degToRad(teapotAngle), [0, 1, 0]); //再做 rotate。 [0,1,0] -> 旋轉軸 x,y,z  teapotAngle
    mat4.rotate(mvMatrix, degToRad(270), [1, 0, 0]); //再做 rotate。 [0,1,0] -> 旋轉軸 x,y,z  teapotAngle
    mat4.scale(mvMatrix, [1,1,1]); // 1就是不變，填放大倍數
    //實際上teapot會先旋轉再平移
    //矩陣相乘的順序會影響結果
    //scale: 改變大小
    setMatrixUniforms();
```

```
function animate() {
    var timeNow = new Date().getTime();
    if (lastTime != 0) {
        var elapsed = timeNow - lastTime;
        teapotAngle += 0.2  * elapsed;  //每次會重新抓時間，加到teapotAngle, 所以每秒的angle都不一樣，看起來就是一直旋轉
    }
    
    lastTime = timeNow;
}
```

```
function tick() {
    requestAnimFrame(tick);
    drawScene();  //對剛才抓出來的所有attribute 跟 matrix 做設定
    animate();
}
```


* 做 transform 的時候， code 由上到下 ，對應 matrix 由左至右。

* function `handleLoadedTeapot` 中， 藉由以下部分讀取 vertex normal

```
teapotVertexPositionBuffer = gl.createBuffer();
gl.bindBuffer(gl.ARRAY_BUFFER, teapotVertexPositionBuffer);
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(teapotData.vertexPositions), gl.STATIC_DRAW);
teapotVertexPositionBuffer.itemSize = 3;
teapotVertexPositionBuffer.numItems = teapotData.vertexPositions.length / 3;
```
  


### Gouraud shading
1. 修改 `vertexShader`
2. 作法：把三角形三頂點的顏色計算出來，中間的顏色是由三頂點顏色內差。
3. 會把內差結果傳給 `fragmentShader`
Q: 為什麼在`vertexShader` 打光，最後出來的結果是Gouraud shading？
A: `vertexShader` 處理的是每個頂點最後會出現在螢幕上的哪個位置（gl_position）。不會知道是要畫出三角形。
`fragmentShader` 處理的是每個pixel的顏色。(gl_fragColor)
在 rendering pipeline中，最先處理 vertex processer，等到 clipper 決定哪些pixel 位在三角形裡面，然後依序用內差法得到三角形內每個pixel的顏色，存在 fragColor並且傳給 fragmentShader，完成render。

所以，以此類推，如何做 Phong shading？
在三角形裡面，用三個頂點的normal 做內差(Rasterizer會自動做內差)，再給每個pixel做顏色計算。

-> 把`vertexShader`的各個變數用 varying的方式傳給 `fragmentShader`，在`fragmentShader`裡面再做三個 light 的計算。




1. 在 `vertexShader` 中 增加一個 attribute:
```
attribute vec3 aVertexNormal; 
```
2. 在 `initShaders` 中提取這個attribute:
```
shaderProgram.vertexNormalAttribute = gl.getAttribLocation(shaderProgram, "aVertexNormal");
gl.enableVertexAttribArray(shaderProgram.vertexNormalAttribute);
```

3. 在 `drawScene` 中設定 Vertex data:
```
// Setup teapot vertex data
gl.bindBuffer(gl.ARRAY_BUFFER, teapotVertexNormalBuffer);
gl.vertexAttribPointer(shaderProgram.vertexNormalAttribute, 
                        teapotVertexNormalBuffer.itemSize, 
                        gl.FLOAT, 
                        false, 
                        0, 
                        0);
```


4. Gouraud shading 會需要有三種光源： amnient + diffuse + specular，在做這三種光源之前，會需要四個單位向量：
    - V: 物體看向相機(眼睛) 的向量 = 相機看向物體的反向。 (-mvMatrix)
    - N: 就是 mvNormal 
    - L: lighting, 從物體看向光源。光源位置 - 物體位置
    - H: 用於 specular light。 是 L + V
    * specular light 中，有一個 alpha 角，是 R 與 V 的夾角，其中 R 是反射角。我們用 H 跟 N 的夾角來近似 alpha 值。

    (講義： halfway vector)
    
    Gouraud shading 的結果，當物體旋轉時，會看到一閃一閃的光，
    原因有二：
    1. 三角形太大了
    2. 因為計算顏色時，是以三角形頂點顏色做內差，所以假如一個 specular 亮點剛好位在三角形中間，那它就不會被畫出來。
    如果改成用 phone shading 就會比較smooth一點。


* 如果要用多盞光源，可以用for 迴圈分別計算三種光源的特性。但要注意的是，ambient light 不管光源有多少盞都是定值，不能多重加總。因為 ambient light 本來就是用來模擬光經過多重反射後的結果。






## Reference
[WebGL interactive models]
http://www.ibiblio.org/e-notes/webgl/models.htm 