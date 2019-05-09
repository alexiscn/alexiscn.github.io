---
layout: post
title:  "Metal实现Instagram的滤镜"
date:   2018-06-15 10:21:10 +0800
tag: 开源项目
---

Instagram有很多很好看的Filter，目前github上也有一些开源项目实现了Instagram的滤镜，基本都是基于[GPUImage](https://github.com/BradLarson/GPUImage)，因为Instagram滤镜本身的实现其实也是基于OpenGL的。

写了一个脚本，将Instagram中对照片处理的滤镜信息导出一个JSON文件，如下：

```javascript
[
  {
    "filter": "IGClarendonVideoFilter",
    "fragmentShader": "// apply lgg curves\n                     texel.r = texture2D(map, vec2(texel.r, 0.5)).r;\n                     texel.g = texture2D(map, vec2(texel.g, 0.5)).g;\n                     texel.b = texture2D(map, vec2(texel.b, 0.5)).b;\n\n                     // darken shadows + saturation\n                     float luma = dot(vec3(0.2126, 0.7152, 0.0722), texel.rgb);\n                     float shadowCoeff = 0.35 * max(0.0, 1.0 - luma);\n                     texel.rgb = mix(texel.rgb, max(vec3(0.0), 2.0 * texel.rgb - 1.0), shadowCoeff);\n                     texel.rgb = mix(texel.rgb, vec3(luma), -0.3);\n\n                     // apply color curves\n                     texel.r = texture2D(map2, vec2(texel.r, 0.5)).r;\n                     texel.g = texture2D(map2, vec2(texel.g, 0.5)).g;\n                     texel.b = texture2D(map2, vec2(texel.b, 0.5)).b;",
    "filterName": "Clarendon",
    "samplers": {
      "map": "Glacial1.png",
      "map2": "Glacial2.png"
    },
    "borderName": "filterBorderPlainWhite.png",
    "vertexShader": "",
    "fullVertexShader": "attribute vec3 a_position; attribute vec2 a_texCoord; uniform mat4 u_contentTransform; uniform mat4 u_texCoordTransform; varying vec2 textureCoordinate; varying vec2 sourceTextureCoordinate; void main() { gl_Position = u_contentTransform * vec4(a_position, 1.0); textureCoordinate = a_texCoord + vec2(0.5); vec4 texel = u_texCoordTransform * vec4(a_texCoord, 0.0, 1.0); sourceTextureCoordinate = texel.xy / texel.w + vec2(0.5); }",
    "fullFragmentShader": "precision mediump float;varying vec2 textureCoordinate; varying vec2 sourceTextureCoordinate;\n uniform float strength; \nuniform sampler2D map; uniform sampler2D map2; \nuniform sampler2D s_texture; void main() { vec4 texel = texture2D(s_texture, sourceTextureCoordinate);vec4 inputTexel = texel;// apply lgg curves\n                     texel.r = texture2D(map, vec2(texel.r, 0.5)).r;\n                     texel.g = texture2D(map, vec2(texel.g, 0.5)).g;\n                     texel.b = texture2D(map, vec2(texel.b, 0.5)).b;\n\n                     // darken shadows + saturation\n                     float luma = dot(vec3(0.2126, 0.7152, 0.0722), texel.rgb);\n                     float shadowCoeff = 0.35 * max(0.0, 1.0 - luma);\n                     texel.rgb = mix(texel.rgb, max(vec3(0.0), 2.0 * texel.rgb - 1.0), shadowCoeff);\n                     texel.rgb = mix(texel.rgb, vec3(luma), -0.3);\n\n                     // apply color curves\n                     texel.r = texture2D(map2, vec2(texel.r, 0.5)).r;\n                     texel.g = texture2D(map2, vec2(texel.g, 0.5)).g;\n                     texel.b = texture2D(map2, vec2(texel.b, 0.5)).b; \ntexel.rgb = mix(inputTexel.rgb, texel.rgb, strength);\n\ngl_FragColor = texel;\n}"
  },
  ....
]
```

其中：

* filter 表示在Instagram中滤镜的类名
* borderName：滤镜对应的背景素材
* fullVertexShader：传给OpenGL的全部VertexShader
* fullFragmentShader：传给OpenGL的全部FragmentShader

我们可以写一个脚本，批量将OpenGL的脚本转换为Metal下的脚本，如vec2在metal下为float2。

```shader
texel.r = texture2D(map, vec2(texel.r, 0.5)).r;
```

翻译为metal为：

```
texel.r = map.sample(s, float2(texel.r, 0.5)).r;
```

由于这些滤镜变化的是FragmentShader，以及使用的采样图片，处理流程都是一样的，所以就可以写一个基类。每一个滤镜都是他的子类，子类中重写滤镜的名字、边框的图片资源、采样图片等等。


```swift
class MTFilter: NSObject, MTIUnaryFilter {

  required override init() { }
    
    // MARK: - Should overrided by subclasses
    class var name: String { return "" }
    
    /// border image Name
    var borderName: String { return "" }

    /// fragment shader name in Metal
    var fragmentName: String { return "" }

    /// Textures, key should match parameter name
    var samplers: [String: String] { return [:] }
    
    var parameters: [String: Any] { return [:] }
    
    /// Strength to adjust filter, ranges in [0.0, 1.0]
    /// if value is 0.0, means no filter effect
    /// if values is 1.0, means full filter effect
    var strength: Float = 1.0

    /// override this function to modifiy samplers if needed
    ///
    /// - Returns: final samplers passes into Metal
    func modifySamplersIfNeeded(_ samplers: [MTIImage]) -> [MTIImage] {
        return samplers
    }

    // MARK: - MTIUnaryFilter
    
    var inputImage: MTIImage?
    
    var outputPixelFormat: MTLPixelFormat = .invalid
}
```

`MTFilter`实现了MetalPetal中的`MTIUnaryFilter`协议，该协议的定义如下：

```swift
@protocol MTIFilter <NSObject>

@property (nonatomic) MTLPixelFormat outputPixelFormat; //Default: MTIPixelFormatUnspecified aka MTLPixelFormatInvalid

@property (nonatomic, readonly, nullable) MTIImage *outputImage;

@end

@protocol MTIUnaryFilter <MTIFilter>

@property (nonatomic, strong, nullable) MTIImage *inputImage;

@end
```


代码 [MetalFilters](https://github.com/alexiscn/MetalFilters) 已开源。