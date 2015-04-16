---
layout:     post
title:      "Generate Immediate Drawings from Ruby with MiniMagick"
date:       2015-04-16 11:30:00
categories: programming ruby
---
Chances are you've used [ImageMagick][imagemagick] at some point, especially if you've ever had to code some form
of user image upload. I'm a fan of the [MiniMagick][minimagick] gem for using ImageMagick from within Ruby. However, something
I haven't played with much (until today) is using ImageMagick to generate images instead of just scaling and cropping them.

If you're using ImageMagick from the command line, you can generate a variety of shapes using the draw command, with syntax
like this (in this example, drawing a line polygon from a set of points).

    convert -size 150x150 xc:#e0e0e0 \
            -draw "fill none stroke red polyline 30,30 60,20 90,59 80,110 65,90 60,95 40,50 20,60 35,45 30,30" \
            output.png

Using MiniMagick's syntax:

{% highlight ruby %}
points = [[30, 30], [60, 20], [90, 59], [80, 110], [65, 90], [60, 95], [40, 50], [20, 60], [35, 45], [30, 30]]

MiniMagick::Tool::Convert.new do |convert|
  convert.size("150x150")
  convert << "xc:#e0e0e0"
  convert.draw("fill none stroke red polyline #{points.map {|p| p.join(',')}.join(' ')}")
  convert << "output.png"
end
{% endhighlight %}

If you're running this in the console, you can immediately open and view the generated polygon just like this (assuming you're running
Mac OSX, anyway -- you might need to substitute a different command for other platforms).

{% highlight ruby %}
  `open output.png`
{% endhighlight %}

---

A potentially useful and cool use case for this is visualizing (even just for debugging purposes) a set of polygon points or map
coordinates. Since this is intended to run from IRB or the Rails console, simply write the image into a temp file and then open it
for display.

{% highlight ruby %}
class Polygon
  # An array of points represented as [x, y]
  attr_accessor :points

  def bounds
    x_values, y_values = points.transpose
    [x_values.min, y_values.min, x_values.max, y_values.max]
  end

  def visualize
    margin = 16 # pixels
    bounds = self.bounds
    width  = bounds[2] - bounds[0] + margin * 2
    height = bounds[3] - bounds[1] + margin * 2

    points_to_draw = points.map {|point|
      [point[0] - bounds[0] + margin, point[1] - bounds[1] + margin]
    }
    points_to_draw << points_to_draw[0]

    filename = Dir::Tmpname.create(['convert', '.png']) {}
    MiniMagick::Tool::Convert.new do |convert|
      convert.size("#{width}x#{height}")
      convert << "xc:#e0e0e0"
      convert.draw("fill none stroke #f0a0a0 stroke-width 3 polyline #{points_to_draw.map {|p| p.join(',')}.join(' ')}")
      convert << filename
    end
    `open #{filename}`
  end
end

p = Polygon.new
p.points = [[30, 30], [60, 90], [100, 20], [140, 50], [200, 80], [280, -10], [210, -25], [40, -50], [20, -30]]
p.visualize
{% endhighlight %}

![Output Image]({{site.baseurl}}/img/example-polygon-output.png)

[imagemagick]: http://www.imagemagick.org/
[minimagick]: https://github.com/minimagick/minimagick

