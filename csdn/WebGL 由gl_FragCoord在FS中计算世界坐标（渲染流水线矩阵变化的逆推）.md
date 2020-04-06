&#8195;&#8195;今天有群友在交流群里问，如何根据gl_FragCoord在片元坐标中计算当前片元的世界坐标，因为能从顶点着色器传递到片元着色器，大家都没有在意这件事情，晚上正好无事也进行了一番探索，这是一个矩阵变换的逆过程。渲染管线中矩阵变换可参考我的这篇文章[Webgl矩阵变换总结](https://blog.csdn.net/weixin_37683659/article/details/79622618)。
&#8195;&#8195;话不多说直接，上效果，估计有点蒙，解释一下，初始是红色立方体，然后过两秒，开始进行比对，gl_FragCoord在片元着色器中计算世界的坐标和顶点着色器传递到片元着色的世界坐标，如果两者在一定误差范围内，立方体就会变成黄色，下图截取的是立方体的一部分。
![好闪](https://img-blog.csdnimg.cn/20200106225836161.gif)

&#8195;&#8195;上着色器代码，vs和fs。

```c
			precision mediump float;
			precision mediump int;

			uniform mat4 modelViewMatrix; // optional
			uniform mat4 projectionMatrix; // optional

			attribute vec3 position;

			varying vec3 vPosition;

			void main()	{

				vPosition = (modelViewMatrix * vec4( position, 1.0 )).xyz;

				gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );

			}
```

&#8195;&#8195;顶点着色器部分比较简单，就是计算gl_Position和vPosition ，vPosition是相机空间的坐标，不是世界空间的坐标，偷个懒，变换到相机空间，也就能变换世界空间了，原理都一样。

```c
			precision mediump float;
			precision mediump int;

			//投影矩阵的逆矩阵
      		uniform mat4 projectionMatrixInverse;
      		//视口变换矩阵的逆矩阵
			uniform mat4 viewPortMatrixInverse;

			//开始测试的标志
			uniform bool testFlag;

			varying vec3 vPosition;

			//在0.00001的误差范围内进行比较
			bool compare(float valueA,float  valueB){

        		if(valueA>(valueB-0.00001) && valueA<(valueB+0.00001)){
          			return true;
        		}else{
          			return false;
        		}

			}

			void main()	{

                //屏幕坐标系变换到NDC标准设备空间
				vec4 ndcPosition = viewPortMatrixInverse * vec4(gl_FragCoord.xyz,1.0);

                //标准设备空间变换到剪裁空间
				vec4 clipPosition = ndcPosition/gl_FragCoord.w;

				//剪裁空间变换到相机空间
				vec4 viewPosition = projectionMatrixInverse * clipPosition;

				//vs传递过来相机空间的顶底坐标和通过gl_FragCoord算出的相机空间进行比对
				if(	compare(viewPosition.x,vPosition.x)&&
					compare(viewPosition.y,vPosition.y)&&
					compare(viewPosition.z,vPosition.z)&& testFlag){

					//黄色，计算正确显示
					gl_FragColor=vec4(1.0,1.0,0.0,1.0);

				}else{

					//红色，默认显示
					gl_FragColor=vec4(1.0,0.0,0.0,1.0);
				}

			}
```
&#8195;&#8195;片元着色器中，主要对通过gl_FragCoord计算得出的相机空间坐标和顶点着色器传递过来的相机空间坐标进行比对，如果误差在一定范围内就认为计算正确。加上误差范围主要是因为浮点数误差。
&#8195;&#8195;简单说一下，如何通过gl_FragCoord计算相机空间的坐标，gl_FragCoord是Webgl的内置变量，官方文档中这么形容。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020010623162113.png)
&#8195;&#8195;简单来说，x,y就是屏幕viewport中的坐标，z就是变换到（0,1）的片元深度值，1/w就是透视除法时候,用到的w。那如何计算呢。计算流程是这样的，屏幕坐标系->ndc标准设备坐标系（三个轴都在<-1,1>的范围内）->剪裁坐标系->相机坐标系，每一步都乘对应的逆矩阵进行变换，透视除法的逆过程就是乘上w,也就是除gl_FragCoord.w，代码中已经写明注释，可以自己想一想。
&#8195;&#8195;我是基于Three.js进行开发的，viewPortMatrixInverse可能并不是很好获取，我给出代码，后期我在说明推导过程，先给出关键的代码。

```javascript
function getViewPortMatrixInverse(viewPort, depthRange) {

		var _viewPortMatrix = new THREE.Matrix4();
		var _viewPortMatrixInverse = new THREE.Matrix4();

		_viewPortMatrix.set(
				viewPort.width / 2, 0, 0, viewPort.x + viewPort.width / 2,
				0, viewPort.height / 2, 0, viewPort.y + viewPort.height / 2,
				0, 0, (depthRange.far - depthRange.near) / 2, (depthRange.far + depthRange.near) / 2,
				0, 0, 0, 1
		);

		_viewPortMatrixInverse.getInverse(_viewPortMatrix, true);

		return _viewPortMatrixInverse;
	}
```
&#8195;&#8195;然后再给出mesh的构建过程代码。

```javascript
		var geometry = new THREE.BoxBufferGeometry();

		viewPortMatrixInverse = getViewPortMatrixInverse(
				{x: 0, y: 0, width: window.innerWidth, height: window.innerHeight},
				{near: 0, far: 1}
		);
		camera.updateProjectionMatrix();

		material = new THREE.RawShaderMaterial({

			uniforms: {
				projectionMatrixInverse: {value: camera.projectionMatrixInverse},
				viewPortMatrixInverse: {value: viewPortMatrixInverse},
				testFlag: {value: false}
			},
			vertexShader: document.getElementById('vertexShader').textContent,
			fragmentShader: document.getElementById('fragmentShader').textContent,
			side: THREE.DoubleSide,
			transparent: true
		});

		var mesh = new THREE.Mesh(geometry, material);
```
&#8195;&#8195;好了，到这里就结束了，该洗洗睡了，可能写的漏洞百出，请各位大佬多指点，可以加我的技术交流群指点，嘻嘻，都是做3D的同志，webgl,opengl,vulkan，unity,ue4等等。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200106232941840.png)
&#8195;&#8195;忘了说testFlag，这个的作用就是为了显示计算结果的正确性，就是这样。

```javascript
setTimeout(function () {
		material.uniforms["testFlag"].value = true;
	}, 2000);
```
&#8195;&#8195;睡觉Zzzzzzzz。
