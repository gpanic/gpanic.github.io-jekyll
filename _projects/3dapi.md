---
layout: default
page_title: 3dApi
type: personal
year: 2015
platform: Windows
role: Programmer
tools: C++, Direct3D, OpenGL, HLSL, GLSL, Visual Studio, Blender
release: 2015
website: https://github.com/gpanic/3dApi
description: 3dApi is a real time rendering engine used to benchmark Direct3D and OpenGL.
image_path: images/project_thumb_3dapi.jpg
---
<div class="project-page">
	<div class="description">
		<div class="container">
			<h1>3dApi</h1>
			<div class="row detail">
				<div class="col-md-6 first">Platform</div>
				<div class="col-md-6 second">{{ page.platform }}</div>
			</div>
			<div class="row detail">
				<div class="col-md-6 first">Role</div>
				<div class="col-md-6 second">{{ page.role }}</div>
			</div>
			<div class="row detail">
				<div class="col-md-6 first">Tools used</div>
				<div class="col-md-6 second">{{ page.tools }}</div>
			</div>
			<div class="row detail">
				<div class="col-md-6 first">Released</div>
				<div class="col-md-6 second">{{ page.release }}</div>
			</div>
			<div class="row detail">
				<div class="col-md-6 first">Source code</div>
				<div class="col-md-6 second"><a href="{{ page.website }}">{{ page.website }}</a></div>
			</div>
		</div>
	</div>
	<div class="media">
		<div class="container">
			<div id="carousel-example-generic" class="carousel slide" data-ride="carousel" data-interval="false">
				<ol class="carousel-indicators">
					<li data-target="#carousel-example-generic" data-slide-to="0" class="active"></li>
					<li data-target="#carousel-example-generic" data-slide-to="1"></li>
					<li data-target="#carousel-example-generic" data-slide-to="2"></li>
					<li data-target="#carousel-example-generic" data-slide-to="3"></li>
				</ol>

				<div class="carousel-inner" role="listbox">
					<div class="item active">
						<img src="/images/screenshots/3dapi_1.jpg" />
					</div>
					<div class="item">
						<img src="/images/screenshots/3dapi_2.jpg" />
					</div>
					<div class="item">
						<img src="/images/screenshots/3dapi_3.jpg" />
					</div>
					<div class="item">
						<img src="/images/screenshots/3dapi_4.jpg" />
					</div>
				</div>
			</div>
		</div>
	</div>
	<div class="text">
		<div class="container">
			<p>
				3dApi is a real time rendering engine used to benchmark Direct3D and OpenGL. It includes several scenes that use different rendering techniques. They are all implemented as indentical as possible in both APIs, then benchmarked to compare the performance and resulting images.
			</p>
			<p>
				3dApi was developed as the practical part of my master's thesis. The goal of the thesis was to compare Direct3D and OpenGL and highlight the differences and similarities. This comparison included aspects such as history, fields of use, architecture, features, conventions, ease of use and performance. By completing the thesis I gained a deeper theoretical and practical understanding of the real time graphics pipeline. I learned the basics of the two most widely used 3d graphics APIs, shading languages and a lot of techniques used in real time rendering.
			</p>
			<b>Highlighted features:</b>
			<ul>
				<li>Real time rendering engine in both Direct3D and OpenGL</li>
				<li>Framework for quickly creating applications in both APIs</li>
				<li>Phong lighting model</li>
				<li>Directional lights</li>
				<li>Shadowmapping</li>
			</ul>
		</div>
	</div>
</div>