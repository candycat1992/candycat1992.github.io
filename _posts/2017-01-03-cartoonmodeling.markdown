---
title:      "An Interactive 2D-to-3D Cartoon Modeling System"
layout:     default
date:       2017-01-02 00:00:01
author:     "Candycat"
header-img: "img/in-post/2017-01-03-cartoonmodeling/cartoon-modeling-title.jpg"
tags:
    - Research
---

<!-- Page Header -->
<header class="intro-header" style="background-image: url('{{ site.baseurl }}/{% if page.header-img %}{{ page.header-img }}{% else %}{{ site.header-img }}{% endif %}')">
    <div class="container">
        <div class="row">
            <div class="col-lg-8 col-lg-offset-2 col-md-10 col-md-offset-1">
                <div class="site-heading" id="tag-heading">
                    <h1><font size="8">An Interactive 2D-to-3D Cartoon Modeling System</font></h1>
                </div>
            </div>
        </div>
    </div>
</header>

<head>

<style type="text/css">

h2,h3,h4,h5,h6 {
    font-family:'Open Sans',sans-serif;
    color:#0085a1;
    text-rendering:optimizeLegibility;
    margin-top:.2em;
    margin-bottom:.5em;
    line-height:1.2125em;
    font-weight:400;
    font-style:normal
}

h2 {
    font-size:1.6875em
}

h3 {
    font-size:1.375em
}

h4 {
    font-size:1.125em
}

h5 {
    font-size:1.125em
}

h6 {
    font-size:1em
}
p {
	font-size: 16px;
	margin-top: 5px;
	text-align: justify;
	line-height: 30px;
}

pre.bibtex {
 white-space: pre-wrap;       /* css-3 */
 white-space: -moz-pre-wrap;  /* Mozilla, since 1999 */
 white-space: -pre-wrap;      /* Opera 4-6 */
 white-space: -o-pre-wrap;    /* Opera 7 */
 word-wrap: break-word;       /* Internet Explorer 5.5+ */
}

#publication-papers {
	display: block;
	float: left;
	list-style: none;
	margin: 20px 0 20px 0;
	padding: 0;
	text-align: center;
	width: 1020px;
}

#publication-papers li {
	background: url('{{ site.baseurl }}/img/toolbox/paper-pdf.png') 0 0 no-repeat;
	display: inline-block;
	font: normal 15pt/16pt 'Trebuchet MS', sans-serif;
	padding: 2px 2px 4px 37px;
	margin: 0 7px;
	min-height: 24px;
}

#publication-papers li a {
	/*border-bottom: 1px dotted #00bce9;*/
	color: #0060af;
	text-decoration: none;
}

#publication-papers li a:hover {
	/*border-bottom: none;*/
	color: #00bce9;
}

#publication-attachments li {
	display: block;
	float: left;
	font: normal 11pt/16pt 'Trebuchet MS', sans-serif;
	height: 29px;
	margin: 0 0 10px 0;
	overflow: hidden;
	padding: 3px 0 0 40px;
	width: 164px;
}

#publication-attachments li.jpg { background: url('{{ site.baseurl }}/img/toolbox/filetypes/jpg.png') 0 0 no-repeat; }
#publication-attachments li.pdf { background: url('{{ site.baseurl }}/img/toolbox/filetypes/pdf.png') 0 0 no-repeat; }
#publication-attachments li.png { background: url('{{ site.baseurl }}/img/toolbox/filetypes/png.png') 0 0 no-repeat; }
#publication-attachments li.rar { background: url('{{ site.baseurl }}/img/toolbox/filetypes/rar.png') 0 0 no-repeat; }
#publication-attachments li.txt { background: url('{{ site.baseurl }}/img/toolbox/filetypes/txt.png') 0 0 no-repeat; }
#publication-attachments li.zip { background: url('{{ site.baseurl }}/img/toolbox/filetypes/zip.png') 0 0 no-repeat; }
#publication-attachments li.htm { background: url('{{ site.baseurl }}/img/toolbox/filetypes/htm.png') 0 0 no-repeat; }

#publication-attachments li img {
	vertical-align: middle;
}

#publication-attachments li a {
	border-bottom: 1px dotted #00bce9;
	color: #0060af;
	margin: 0;
	text-decoration: none;
}

#publication-attachments li a:hover {
	border-bottom: none;
	color: #00bce9;
}

</style>

</head>

<!-- Main Content -->
<div class="container">
	<div class="row">
        <div class="col-lg-8 col-lg-offset-2 col-md-10 col-md-offset-1">
			<h4>Authors</h4>
			<p>
				Lele Feng, Xubo Yang, Shuangjiu Xiao, Fan Jiang
			</p>
						
			<h4>Publication</h4>
			<p>
				<a href="http://link.springer.com/book/10.1007/978-3-319-40259-8" target="_blank">10th International Conference, Edutainment 2016, Hangzhou, China, April, 2016</a>
			</p>

			<h4>Abstract</h4>
	         <p>In this paper, we propose an interactive system that can quickly convert a 2D cartoon painting into a 3D textured cartoon model, enabling non-professional adults and children to easily create personalized 3D contents. Our system exploits a new approach based on solving Poisson equations to generate 3D models, which is free from the limitations of spherical topology in prior works. We also propose a novel method to generate whole textures for both sides of the models to deliver colorful appearances, making it possible to obtain stylized models rendered with cartoon textures. The results have shown that our method can greatly simplify the modeling process comparing with both traditional modeling softwares and prior sketch-based systems.</p>
			
			<h4>Videos</h4>
            <p><iframe width="560" height="315" src="https://www.youtube.com/embed/NystYCMcVDI" frameborder="0" allowfullscreen></iframe></p>

			<h4>BibTeX</h4>
			<p>
			<pre class="bibtex">
@inproceedings{feng2016an,
title={An interactive 2D-to-3D cartoon modeling system},
author={Feng, Lele and Yang, Xubo and Xiao, Shuangjiu and Jiang, Fan},
booktitle={The 10th International Conference on E-Learning and Games (Edutainment 2016)},
year={2016}
address = {Hangzhou, CN}}
			</pre>
			</p>

			<h4>Attachments</h4>
	        <ul id="publication-attachments">
	            <li class="pdf"><a href="http://candycat1992.github.io/attach/papers/2016-cartoon-modeling-system.pdf">Paper (1.7MB)</a></li>
	        </ul>
		</div>
	</div>
</div>