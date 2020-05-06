# webGL_practice
This practice is based on Homework 1 of ICG@NTU. 

## Goal
* Flat, Gouraud, and Phong shading with Phong reflection model in shaders.
* Multiple transformations (four fundamental transforms) on objects in a scene.
* At least 3 objects & at least 3 light sources


## Code Notes

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



function animate() {
    var timeNow = new Date().getTime();
    if (lastTime != 0) {
        var elapsed = timeNow - lastTime;
        teapotAngle += 0.2  * elapsed;  //每次會重新抓時間，加到teapotAngle, 所以每秒的angle都不一樣，看起來就是一直旋轉
    }
    
    lastTime = timeNow;
}



function tick() {
    requestAnimFrame(tick);
    drawScene();  //對剛才抓出來的所有attribute 跟 matrix 做設定
    animate();
}



做 transform 的時候， code 由上到下 ，對應 matrix 由左至右。

function `handleLoadedTeapot` 中， 藉由以下部分讀取 vertex normal


    teapotVertexPositionBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, teapotVertexPositionBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(teapotData.vertexPositions), gl.STATIC_DRAW);
    teapotVertexPositionBuffer.itemSize = 3;
    teapotVertexPositionBuffer.numItems = teapotData.vertexPositions.length / 3;

  


### Gouraud shading
1. 修改 `vertexShader`
2. 作法：把三角形三頂點的顏色計算出來，中間的顏色是由三頂點顏色內差。
3. 會把內差結果傳給 `fragmentShader`
Q: 為什麼在`vertexShader` 打光，最後出來的結果是Gouraud shading？


1. 在 `vertexShader` 中 增加一個 attribute:
''' 
attribute vec3 aVertexNormal; 
'''
