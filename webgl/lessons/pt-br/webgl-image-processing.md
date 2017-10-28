Title: Processamento de imagem WebGL2
Description: Como processar imagens na WebGL

O processamento de imagens é fácil na WebGL. Quão fácil? Leia abaixo.

Esta é uma continuação de [Fundamentos da WebGL2](webgl-fundamentals.html).
Se você não leu, eu sugiro [ir lá primeiro](webgl-fundamentals.html).

Para desenhar imagens na WebGL precisamos usar texturas. Da mesma forma que a
WebGL espera coordenadas de clippace ao renderizar em vez de pixels,
WebGL geralmente espera coordenadas de textura ao ler uma textura.
As coordenadas da textura variam de 0,0 a 1,0 independentemente das dimensões da textura.

WebGL2 adiciona a capacidade de ler uma textura usando coordenadas de pixels também.
Qual caminho é melhor depende de você. Sinto que é mais comum usar as 
coordenadas de textura do que as coordenadas de pixels.

Uma vez que estamos apenas desenhando um único retângulo (bem, 2 triângulos),
precisamos dizer a WebGL qual lugar na textura em que cada ponto do
retângulo corresponde. Vamos passar esta informação do vertex
para o fragmento shader usando um tipo especial de variável chamado
de 'varying'. É chamado de varying porque isso varia. [WebGL irá
interpolar os valores](webgl-how-it-works.html) que fornecemos no
vertex shader pois desenha cada pixel usando o fragmento shader.

Usando [o vertex shader do final da publicação anterior](webgl-fundamentals.html)
precisamos adicionar um atributo para passar em coordenadas de textura e depois
passar para o fragmento shader.

    ...

    +in vec2 a_texCoord;

    ...

    +out vec2 v_texCoord;

    void main() {
       ...
    +   // passe o texCoord para o fragmento shader
    +   // O GPU irá interpolar esse valor entre pontos
    +   v_texCoord = a_texCoord;
    }

Então, fornecemos um fragmento shader para procurar cores da textura.

    #version 300 es
    precision mediump float;

    // nossa textura
    uniform sampler2D u_image;

    // O texCoords passou do vertex shader.
    in vec2 v_texCoord;

    // precisamos declarar uma saída para o fragmento shader
    out vec4 outColor;

    void main() {
       // Procure uma cor da textura.
       outColor = texture(u_image, v_texCoord);
    }

Finalmente, precisamos carregar uma imagem, criar uma textura e copiar a imagem
para a textura. Como estamos em imagens de um navegador, carregamos de forma assíncrona,
então devemos reorganizar nosso código um pouco para aguardar o carregamento da textura.
Uma vez que carregada, vamos desenhá-la.

    +function main() {
    +  var image = new Image();
    +  image.src = "http://someimage/on/our/server";  // DEVE SER MESMO DOMÍNIO!!!
    +  image.onload = function() {
    +    render(image);
    +  }
    +}

    function render(image) {
      ...
      // procure onde os dados do vértice precisam ir.
      var positionAttributeLocation = gl.getAttribLocation(program, "a_position");
    +  var texCoordAttributeLocation = gl.getAttribLocation(program, "a_texCoord");

      // uniformes de pesquisa
      var resolutionLocation = gl.getUniformLocation(program, "u_resolution");
    +  var imageLocation = gl.getUniformLocation(program, "u_image");

      ...

    +  // fornecer coordenadas de textura para o retângulo.
    +  var texCoordBuffer = gl.createBuffer();
    +  gl.bindBuffer(gl.ARRAY_BUFFER, texCoordBuffer);
    +  gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([
    +      0.0,  0.0,
    +      1.0,  0.0,
    +      0.0,  1.0,
    +      0.0,  1.0,
    +      1.0,  0.0,
    +      1.0,  1.0]), gl.STATIC_DRAW);
    +  gl.enableVertexAttribArray(texCoordAttributeLocation);
    +  var size = 2;          // 2 componentes por iteração
    +  var type = gl.FLOAT;   // os dados são floats de 32bit 
    +  var normalize = false; // não normalize os dados
    +  var stride = 0;        // 0 = mover para o tamanho * sizeof (tipo) cada iteração para obter a próxima posição
    +  var offset = 0;        // comece no início do buffer
    +  gl.vertexAttribPointer(
    +      texCoordAttributeLocation, size, type, normalize, stride, offset)
    +
    +  // faça da unidade 0 a unidade de textura ativa
    +  // (ie, the unit all other texture commands will affect
    +  gl.activeTexture(gl.TEXTURE0 + 0);
    +
    +  // Vincule a unidade de textura 0' ponto de ligação 2D
    +  gl.bindTexture(gl.TEXTURE_2D, texture);
    +
    +  // Defina os parâmetros para que não precisemos de mips e por isso não estamos filtrando
    +  // e não repetindo
    +  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    +  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    +  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
    +  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
    +
    +  // Carregue a imagem para a textura.
    +  var mipLevel = 0;               // the largest mip
    +  var internalFormat = gl.RGBA;   // format we want in the texture
    +  var srcFormat = gl.RGBA;        // format of data we are supplying
    +  var srcType = gl.UNSIGNED_BYTE  // type of data we are supplying
    +  gl.texImage2D(gl.TEXTURE_2D,
    +                mipLevel,
    +                internalFormat,
    +                srcFormat,
    +                srcType,
    +                image);

      ...

      // Diga para usar nosso programa (par de shaders)
      gl.useProgram(program);

      // Passe na resolução da tela para que possamos converter
      // pixels para clipspace no shader
      gl.uniform2f(resolutionLocation, gl.canvas.width, gl.canvas.height);

    +  // Diga ao shader para obter a textura da unidade de textura 0
    +  gl.uniform1i(imageLocation, 0);

    +  // Vincule o buffer de posição para que gl.bufferData seja chamado
    +  // em setRectangle para colocar dados no buffer de posição
    +  gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
    +
    +  // Defina um retângulo do mesmo tamanho que a imagem.
    +  setRectangle(gl, 0, 0, image.width, image.height);

    }

And here's the image rendered in WebGL.

{{{example url="../webgl-2d-image.html" }}}

Not too exciting so let's manipulate that image. How about just
swapping red and blue?

    ...
    outColor = texture2D(u_image, v_texCoord).bgra;
    ...

And now red and blue are swapped.

{{{example url="../webgl-2d-image-red2blue.html" }}}

What if we want to do image processing that actually looks at other
pixels? Since WebGL references textures in texture coordinates which
go from 0.0 to 1.0 then we can calculate how much to move for 1 pixel
 with the simple math <code>onePixel = 1.0 / textureSize</code>.

Here's a fragment shader that averages the left and right pixels of
each pixel in the texture.

```
#version 300 es

// fragment shaders don't have a default precision so we need
// to pick one. mediump is a good default. It means "medium precision"
precision mediump float;

// our texture
uniform sampler2D u_image;

// the texCoords passed in from the vertex shader.
in vec2 v_texCoord;

// we need to declare an output for the fragment shader
out vec4 outColor;

void main() {
+  vec2 onePixel = vec2(1) / vec2(textureSize(u_image, 0));
+
+  // average the left, middle, and right pixels.
+  outColor = (
+      texture(u_image, v_texCoord) +
+      texture(u_image, v_texCoord + vec2( onePixel.x, 0.0)) +
+      texture(u_image, v_texCoord + vec2(-onePixel.x, 0.0))) / 3.0;
}
```

Compare to the un-blurred image above.

{{{example url="../webgl-2d-image-blend.html" }}}

Now that we know how to reference other pixels let's use a convolution kernel
to do a bunch of common image processing. In this case we'll use a 3x3 kernel.
A convolution kernel is just a 3x3 matrix where each entry in the matrix represents
how much to multiply the 8 pixels around the pixel we are rendering. We then
divide the result by the weight of the kernel (the sum of all values in the kernel)
or 1.0, whichever is greater. [Here's a pretty good article on it](http://docs.gimp.org/en/plug-in-convmatrix.html).
And [here's another article showing some actual code if
you were to write this by hand in C++](http://www.codeproject.com/KB/graphics/ImageConvolution.aspx).

In our case we're going to do that work in the shader so here's the new fragment shader.

```
#version 300 es

// fragment shaders don't have a default precision so we need
// to pick one. mediump is a good default. It means "medium precision"
precision mediump float;

// our texture
uniform sampler2D u_image;

// the convolution kernal data
uniform float u_kernel[9];
uniform float u_kernelWeight;

// the texCoords passed in from the vertex shader.
in vec2 v_texCoord;

// we need to declare an output for the fragment shader
out vec4 outColor;

void main() {
  vec2 onePixel = vec2(1) / vec2(textureSize(u_image, 0));

  vec4 colorSum =
      texture(u_image, v_texCoord + onePixel * vec2(-1, -1)) * u_kernel[0] +
      texture(u_image, v_texCoord + onePixel * vec2( 0, -1)) * u_kernel[1] +
      texture(u_image, v_texCoord + onePixel * vec2( 1, -1)) * u_kernel[2] +
      texture(u_image, v_texCoord + onePixel * vec2(-1,  0)) * u_kernel[3] +
      texture(u_image, v_texCoord + onePixel * vec2( 0,  0)) * u_kernel[4] +
      texture(u_image, v_texCoord + onePixel * vec2( 1,  0)) * u_kernel[5] +
      texture(u_image, v_texCoord + onePixel * vec2(-1,  1)) * u_kernel[6] +
      texture(u_image, v_texCoord + onePixel * vec2( 0,  1)) * u_kernel[7] +
      texture(u_image, v_texCoord + onePixel * vec2( 1,  1)) * u_kernel[8] ;
  outColor = vec4((colorSum / u_kernelWeight).rgb, 1);
}
```

In JavaScript we need to supply a convolution kernel and its weight

     function computeKernelWeight(kernel) {
       var weight = kernel.reduce(function(prev, curr) {
           return prev + curr;
       });
       return weight <= 0 ? 1 : weight;
     }

     ...
     var kernelLocation = gl.getUniformLocation(program, "u_kernel[0]");
     var kernelWeightLocation = gl.getUniformLocation(program, "u_kernelWeight");
     ...
     var edgeDetectKernel = [
         -1, -1, -1,
         -1,  8, -1,
         -1, -1, -1
     ];

    // set the kernel and it's weight
     gl.uniform1fv(kernelLocation, edgeDetectKernel);
     gl.uniform1f(kernelWeightLocation, computeKernelWeight(edgeDetectKernel));
     ...

And voila... Use the drop down list to select different kernels.

{{{example url="../webgl-2d-image-3x3-convolution.html" }}}

I hope this article has convinced you image processing in WebGL is pretty simple. Next up
I'll go over [how to apply more than one effect to the image](webgl-image-processing-continued.html).

<div class="webgl_bottombar">
<h3>What are texture units?</h3>
When you call <code>gl.draw???</code> your shader can reference textures. Textures are bound
to texture units. While the user's machine might support more all WebGL2 implementations are
required to support at least 16 texture units. Which texture unit each sampler uniform
references is set by looking up the location of that sampler uniform and then setting the
index of the texture unit you want it to reference.

For example:
<pre class="prettyprint showlinemods">
var textureUnitIndex = 6; // use texture unit 6.
var u_imageLoc = gl.getUniformLocation(
    program, "u_image");
gl.uniform1i(u_imageLoc, textureUnitIndex);
</pre>

To set textures on different units you call gl.activeTexture and then bind the texture you want on that unit. Example

<pre class="prettyprint showlinemods">
// Bind someTexture to texture unit 6.
gl.activeTexture(gl.TEXTURE6);
gl.bindTexture(gl.TEXTURE_2D, someTexture);
</pre>

This works too

<pre class="prettyprint showlinemods">
var textureUnitIndex = 6; // use texture unit 6.
// Bind someTexture to texture unit 6.
gl.activeTexture(gl.TEXTURE0 + textureUnitIndex);
gl.bindTexture(gl.TEXTURE_2D, someTexture);
</pre>
</div>

<div class="webgl_bottombar">
<h3>What's with the a_, u_, and v_ prefixes in from of variables in GLSL?</h3>
<p>
That's just a naming convention. They are not required but for me it makes it easier to see at a glance
where the values are coming from. a_ for attributes which is the data provided by buffers. u_ for uniforms
which are inputs to the shaders, v_ for varyings which are values passed from a vertex shader to a
fragment shader and interpolated (or varied) between the vertices for each pixel drawn.
See <a href="webgl-how-it-works.html">How it works</a> for more details.
</p>
</div>

