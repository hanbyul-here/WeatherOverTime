![](imgs/00.gif)

## Visualizing Weather over time with Tangram.js

This [Tangram JS](https://github.com/tangrams/tangram) example is the continuation of [a tutorial about 3rd party APIs](https://github.com/tangrams/WeatherNow). In this case, we are gathering the data from [US NOAA's Weather Stations](http://weather.noaa.gov/pub/data/observations/metar/cycles) using a [python script](https://github.com/tangrams/WeatherOverTime/blob/gh-pages/data/getToday.py) on a [RaspberryPi](https://www.raspberrypi.org/), encoding "big data" into an image. We then we display and animate it using shaders inside [Tangram JS](https://github.com/tangrams/tangram).

### How does it work?

![](imgs/rpi.jpg)

Each day over the last two months [this script](https://github.com/tangrams/WeatherOverTime/blob/gh-pages/data/getToday.py) running on a [RaspberryPi](https://www.raspberrypi.org/) has gathered the last 24 hours of weather data.

```bash
python data/getToday.py
```

It then [progressively builds an image](https://github.com/tangrams/WeatherOverTime/blob/gh-pages/data/parseAll.py) containing all the information over the last two months. In it, the temperature, wind speed and wind direction of each US NOAA station is encoded into a single pixel. 

```bash
python data/parseAll.py
```

The `RED` channel contains the temperature for each hour. Wind speed is in the `GREEN` channel, and `BLUE` stores wind direction.

```python
def getColorFor(data, hour, station):
    if data[station].has_key(hour):
        datum = data[station][hour]
        return (int(toFahrenheit(datum['temp'])*2.55), int(datum['wind_speed']*5), int((datum['wind_deg']/360)*255), 255)
    else:
        return None
```

Each station will have row in the final image, while each column will be a single sample per hour over the last two months.

![](data/data.png)

Looking closely to the previous image, the amount of red confirms we are moving in time (X axis), varying between day and night, forming bumps or waves. Also we can differentiate reddish clusters of columns (hot weeks) from greenish ones (cooler weeks). In parallel we can see specific stations (more constant blue or green rows) located in cold places where the temperature is consistently low. 

The presence of wind will create more green or yellow pixels. We can see them come and go with some coherence.

### Why encode data into images?

Encoding big databases into images allows us to compress more than 300Mb of data into a single image of around 3Mb. Also in this format, data can be uploaded to the graphic card unit (GPU) in a single call of the the WebGL/OpenGL driver.  Once it is in the GPU memory, the data can be retrieved, processed and animated at high speed.

### Loading this data to Tangram JS

Once the data is encoded into the image, it can be loaded to Tangram map through the [```scene.yaml```](https://github.com/tangrams/WeatherOverTime/blob/gh-pages/scene.yaml#L21-L24) file:

```yaml
...
textures:
    data_image:
        url: data/data.png
        filtering: nearest
...
```

Together with a [GeoJSON file containing the position of each station](https://github.com/tangrams/WeatherOverTime/blob/gh-pages/scene.yaml#L15-L17) as a POIs...

```YAML
sources:
    ...
    stations:
        type: GeoJSON
        url:   data/stations.geojson
```

...we filter these POIs into a ```layer``` to be rendered with a custom ```style```. We also assign a specific color to each point depending in the ```id``` of the feature.

```YAML
layers:
    ...
    station:
        data: { source: stations }            
        draw:
            wind:
                interactive: true
                collide: false
                blend: overlay
                blend_order: 1
                size: 50
                color: |
                    function() {
                        var hovered = feature.id === global.hovered ? 1 : 0;
                        return [ ( Math.floor(feature.id/255))/255, (feature.id%255)/255, hovered ]; 
                    }
        ...
```

As you can see in the last lines, each ```feature.id``` is encoded into the geometry color of each point, together with a flag that checks if the station has been selected or not (`BLUE` channel).

The custom style for each station point will be rendered with the following code that draws an arrow. The arrow is rotated, scaled and colored according the wind direction, wind speed and temperature.

```YAML
styles:
    ...
    wind:
        base: points
        texcoords: true
        animated: true
        mix: [functions-aastep, geometry-matrices, shapes-cross]
        shaders:
            defines:
                TWO_PI: 6.283185307
                PI: 3.1415926535897932384626433832795
                HALF_PI: 1.5707963267948966192313216916398
            uniforms:
                u_data: data_image
                u_scale: scale_image
                u_param: [812, 1599]
                u_offset: 0
            blocks: 
                global: |
                    float getIndex (vec2 encodedIndex) {
                        return (encodedIndex.x*65025.+encodedIndex.y*255.)+.5;
                    }
                    
                    float shape(vec2 st, int N){
                        st = st *2.-1.;
                        float a = atan(st.x,st.y)+PI;
                        float r = TWO_PI/float(N);
                        return cos(floor(.5+a/r)*r-a)*length(st);
                    }

                    vec4 getColor (vec2 st) {
                        float texWidth = u_param.x;
                        float texHeigth = u_param.y;
                        st.y = 1.-st.y/texHeigth;
                        return texture2D(u_data,vec2(st.x/texWidth,st.y));
                    }
                color: |
                    float index = getIndex(color.st);
                    float t = u_offset;
                    // t += fract(u_time);
                    
                    color = vec4(0.);
                    float i_now = floor(t);
                    float f_now = fract(t);
                    float next = i_now+1.;
                    vec4 data_now = getColor(vec2(mod(i_now,u_resolution.x),index));
                    vec4 data_next = getColor(vec2(mod(next,u_resolution.x),index));
                    vec4 data_interpol = mix(data_now,data_next,f_now);
                    float dir_now = data_now.b;
                    float dir_next = (data_next.g == 0.)? data_now.b : data_next.b;
                    float dir_delta = fract(dir_next-dir_now);
                    if (dir_delta > .5) {
                        dir_delta -= 1.;
                    }
                    dir_next = dir_now + dir_delta;
                    float w_deg = mix(dir_now, dir_next, f_now)*-TWO_PI;
                    float w_speed = data_interpol.g;
                    float temp = data_interpol.r;
                    
                    vec2 st = v_texcoord.xy;
                    float valid = smoothstep(0.5,1.,data_interpol.a);
                    
                    // Arrow
                    float scale = .5;
                    vec2 arrow_st = .5 + (rotate2D(w_deg) * (st - .5));
                    arrow_st *= vec2(scale+.5,.5);
                    float d = shape(arrow_st + vec2(scale*-.5+.25,.3),3) / shape(arrow_st + vec2(scale*-.5+.25,1.132),3);
                    color = texture2D(u_scale,vec2(1.-temp,.5));
                    color.a *= w_speed;
                    color.a = 1.0-step(.1+.15*w_speed,d);
                    // Hovered Arrow
                    color.a -= (1.-step(.05+.05*w_speed,d))*v_color.b;
                    color += max(rect(st,2.)-rect(st,1.6)-cross(st,5.,5.),0.)*v_color.b;
```

Although there's a lot going on in this style, it is important to see how the data is fetched and decoded from the texture. To locate the right pixel, use the function ```getIndex()``` to retrieve the right row (Y axis) where the data for that particular station is located, together with the ```u_offset``` uniform that contains the time offset (X axis).

```glsl
    float index = getIndex(color.st);
    float t = u_offset;
```

Once we have the right positon of the pixel, we can get its interpolated data using ```getColor()``` and decode the values. 

```glsl
    float w_deg = mix(dir_now, dir_next, f_now)*-TWO_PI;
    float w_speed = data_interpol.g;
    float temp = data_interpol.r;
```

**Note**: to correctly interpolate the wind direction, we need to make some extra calculations to smoothly rotate in the shorter direction of the target angle.

Once all the data is ready to be displayed, use the wind direction to rotate the coordinates in the fragment shader, and the wind speed to scale the shape in and out.

```glsl
    // Arrow
    float scale = .5;
    vec2 arrow_st = .5 + (rotate2D(w_deg) * (st - .5));
    arrow_st *= vec2(scale+.5,.5);
    float d = shape(arrow_st + vec2(scale*-.5+.25,.3),3) / shape(arrow_st + vec2(scale*-.5+.25,1.132),3);
    color = texture2D(u_scale,vec2(1.-temp,.5));
    color.a *= w_speed;
    color.a = 1.0-step(.1+.15*w_speed,d);
    // Hovered Arrow
    color.a -= (1.-step(.05+.05*w_speed,d))*v_color.b;
    color += max(rect(st,2.)-rect(st,1.6)-cross(st,5.,5.),0.)*v_color.b;
```

Note how it uses [an extra image with a color gradient ](https://github.com/tangrams/WeatherOverTime/blob/gh-pages/scene.yaml#L25-L26) to "tint" the arrow with a color that expresses the temperature.

![](imgs/scale.png)


## Conclusion

Using [noUISlider component](http://refreshless.com/nouislider/) we can hook up the time offset to a slider and animate it. While the techniques are complex, it pays off with a smooth, flexible and efficient interaction with a very large dataset.

![](imgs/00.gif)
