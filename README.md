LearningSoftRasterization from https://github.com/ssloy/tinyrenderer



# 1 Bresenham’s Line Drawing Algorithm

普通的方法，直接如下按0.01的step进行绘制

```c++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    for (float t=0.; t<1.; t+=1) { 
        int x = x0 + (x1-x0)*t; 
        int y = y0 + (y1-y0)*t; 
        image.set(x, y, color); 
    } 
}
```

则得到的结果如下：

<img src="README_image\image-20220712154504800.png" alt="image-20220712154504800" style="zoom:50%;" />

但是这种方法显然效率极低，还可能存在大量重复绘制。

所以为了避免这种情况还是得按照轴向坐标范围来绘制，如下：

```c++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    for (int x=x0; x<=x1; x++) { 
        float t = (x-x0)/(float)(x1-x0); 
        int y = y0*(1.-t) + y1*t; 
        image.set(x, y, color); 
    } 
}
```

此时结果存在两个问题：

- 如果x0大于x1就不绘制了
- 如果x方向太陡峭，则出现大量断电

<img src="README_image\image-20220712154539445.png" alt="image-20220712154539445" style="zoom:50%;" />

这两个问题可以通过换序解决

```c
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    bool steep = false; 
    if (std::abs(x0-x1)<std::abs(y0-y1)) { // if the line is steep, we transpose the image 
        std::swap(x0, y0); 
        std::swap(x1, y1); 
        steep = true; 
    } 
    if (x0>x1) { // make it left−to−right 
        std::swap(x0, x1); 
        std::swap(y0, y1); 
    } 
    for (int x=x0; x<=x1; x++) { 
        float t = (x-x0)/(float)(x1-x0); 
        int y = y0*(1.-t) + y1*t; 
        if (steep) { 
            image.set(y, x, color); // if transposed, de−transpose 
        } else { 
            image.set(x, y, color); 
        } 
    } 
}
```

此时结果正常了

<img src="README_image\image-20220712155027772.png" alt="image-20220712155027772" style="zoom:50%;" />

但是上面的代码的性能还存在优化空间，可以看到for循环中多算了很多次(x1-x0)和(y1-y0)，可以尝试把这些累积量剥离出来，如下

```c
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    bool steep = false; 
    if (std::abs(x0-x1)<std::abs(y0-y1)) { 
        std::swap(x0, y0); 
        std::swap(x1, y1); 
        steep = true; 
    } 
    if (x0>x1) { 
        std::swap(x0, x1); 
        std::swap(y0, y1); 
    } 
    int dx = x1-x0; 
    int dy = y1-y0; 
    float derror = std::abs(dy/float(dx)); 
    float error = 0; 
    int y = y0; 
    for (int x=x0; x<=x1; x++) { 
        if (steep) { 
            image.set(y, x, color); 
        } else { 
            image.set(x, y, color); 
        } 
        error += derror; 
        if (error>.5) { // 当累积增量大于0.5，则y轴需要跳到下个坐标，error-1
            y += (y1>y0?1:-1); 
            error -= 1.; 
        } 
    } 
} 
```

这种方式减少了大量重复计算，但是美中不足的是，里面包含了float计算，根本原因是除以dx之后又要与0.5比较导致的，那么是不是可以把dx乘回来，

```c
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    bool steep = false; 
    if (std::abs(x0-x1)<std::abs(y0-y1)) { 
        std::swap(x0, y0); 
        std::swap(x1, y1); 
        steep = true; 
    } 
    if (x0>x1) { 
        std::swap(x0, x1); 
        std::swap(y0, y1); 
    } 
    int dx = x1-x0; 
    int dy = y1-y0; 
    int derror2 = std::abs(dy)*2; //此处不要除dx，且乘2避免与0.5比较
    int error2 = 0; 
    int y = y0; 
    for (int x=x0; x<=x1; x++) { 
        if (steep) { 
            image.set(y, x, color); 
        } else { 
            image.set(x, y, color); 
        } 
        error2 += derror2; 
        if (error2 > dx) { // 此处与dx比较即可
            y += (y1>y0?1:-1); 
            error2 -= dx*2; 
        } 
    } 
} 
```

 这就是Bresenham’s Line Drawing Algorithm。



# 2 Triangle rasterization and back face culling

## 2.1 旧扫描线算法

首先可以画这么几个三角形

```c
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    line(t0, t1, image, color); 
    line(t1, t2, image, color); 
    line(t2, t0, image, color); 
}

// ...

Vec2i t0[3] = {Vec2i(10, 70),   Vec2i(50, 160),  Vec2i(70, 80)}; 
Vec2i t1[3] = {Vec2i(180, 50),  Vec2i(150, 1),   Vec2i(70, 180)}; 
Vec2i t2[3] = {Vec2i(180, 150), Vec2i(120, 160), Vec2i(130, 180)}; 
triangle(t0[0], t0[1], t0[2], image, red); 
triangle(t1[0], t1[1], t1[2], image, white); 
triangle(t2[0], t2[1], t2[2], image, green);
```

<img src="README_image\image-20220712161217462.png" alt="image-20220712161217462" style="zoom:50%;" />



所谓的扫描线，顾名思义就是按照指定方向画指定范围的线条，同样要考虑顺序问题

```c
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    // sort the vertices, t0, t1, t2 lower−to−upper (bubblesort yay!) 
    if (t0.y>t1.y) std::swap(t0, t1); 
    if (t0.y>t2.y) std::swap(t0, t2); 
    if (t1.y>t2.y) std::swap(t1, t2); 
    int total_height = t2.y-t0.y; 
    for (int y=t0.y; y<=t1.y; y++) { 
        int segment_height = t1.y-t0.y+1; 
        float alpha = (float)(y-t0.y)/total_height; 
        float beta  = (float)(y-t0.y)/segment_height; // be careful with divisions by zero 
        Vec2i A = t0 + (t2-t0)*alpha; 
        Vec2i B = t0 + (t1-t0)*beta; 
        if (A.x>B.x) std::swap(A, B); 
        for (int j=A.x; j<=B.x; j++) { 
            image.set(j, y, color); // attention, due to int casts t0.y+i != A.y 
        } 
    } 
    for (int y=t1.y; y<=t2.y; y++) { 
        int segment_height =  t2.y-t1.y+1; 
        float alpha = (float)(y-t0.y)/total_height; 
        float beta  = (float)(y-t1.y)/segment_height; // be careful with divisions by zero 
        Vec2i A = t0 + (t2-t0)*alpha; 
        Vec2i B = t1 + (t2-t1)*beta; 
        if (A.x>B.x) std::swap(A, B); 
        for (int j=A.x; j<=B.x; j++) { 
            image.set(j, y, color); // attention, due to int casts t0.y+i != A.y 
        } 
    } 
}
```

<img src="README_image\image-20220712161411877.png" alt="image-20220712161411877" style="zoom:50%;" />

由于两个for循环存在大量相似代码，可以这么合并下

```c
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    if (t0.y==t1.y && t0.y==t2.y) return;
    // sort the vertices, t0, t1, t2 lower−to−upper (bubblesort yay!) 
    if (t0.y>t1.y) std::swap(t0, t1); 
    if (t0.y>t2.y) std::swap(t0, t2); 
    if (t1.y>t2.y) std::swap(t1, t2); 
    int total_height = t2.y-t0.y; 
    for (int i=0; i<total_height; i++) { 
        bool second_half = i>t1.y-t0.y || t1.y==t0.y; 
        int segment_height = second_half ? t2.y-t1.y : t1.y-t0.y; 
        float alpha = (float)i/total_height; 
        float beta  = (float)(i-(second_half ? t1.y-t0.y : 0))/segment_height;
        Vec2i A = t0 + (t2-t0)*alpha; 
        Vec2i B = second_half ? t1 + (t2-t1)*beta : t0 + (t1-t0)*beta; 
        if (A.x>B.x) std::swap(A, B); 
        for (int j=A.x; j<=B.x; j++) { 
            image.set(j, t0.y+i, color); // attention, due to int casts t0.y+i != A.y 
        } 
    } 
}
```

这就是为单线程设计的老扫描线算法。

## 2.2 扫描线算法优化

老的方法需要遍历整个画面，这样如果三角形占用面积较小，则GPU计算时许多Quad是在空跑，更好的办法是算出三角形的AABB包围盒，然后通过判断点是否在三角形内的方法进行绘制，

那么如何判断点是否在三角形内部呢？

对于三角形ABC和点P，其关系可以如下表示

$\overrightarrow{A P}=u \overrightarrow{A C}+v \overrightarrow{A B}$，即 $P=(1-u-v) A+u B+v C$

P点在三角形内部的充分必要条件是：1 >= u >= 0, 1 >= v >= 0, u+v <= 1

解方程:

$$\left\{\begin{array}{r}
u \overrightarrow{A B}_{x}+v \overrightarrow{A C}_{x}+\overrightarrow{P A}_{x}=0 \\
u \overrightarrow{A B}_{y}+v \overrightarrow{A C}_{y}+\overrightarrow{P A}_{y}=0
\end{array}\right.$$

$$\left\{\begin{array}{c}
{\left[\begin{array}{lll}
u & v & 1
\end{array}\right]\left[\begin{array}{l}
\overrightarrow{A B}_{x} \\
\overrightarrow{A C}_{x} \\
\overrightarrow{P A}_{x}
\end{array}\right]=0} \\
{\left[\begin{array}{lll}
u & v & 1
\end{array}\right]\left[\begin{array}{l}
\overrightarrow{A B}_{y} \\
\overrightarrow{A C}_{y} \\
\overrightarrow{P A}_{y}
\end{array}\right]=0}
\end{array}\right.$$$$

即向量[u,v,1]，$[\overrightarrow{A B}_{x},\overrightarrow{A C}_{x},\overrightarrow{P A}_{x}]$, $[\overrightarrow{A B}_{y},\overrightarrow{A C}_{y},\overrightarrow{P A}_{y}]$ 间两两垂直，那么就能如下barycentric函数中求解得到uv

```c
Vec3f barycentric(Vec2i *pts, Vec2i P) { 
    Vec3f u = Vec3f(pts[2].x-pts[0].x, pts[1].x-pts[0].x, pts[0].x-P.x) ^
        Vec3f(pts[2].y-pts[0].y, pts[1].y-pts[0].y, pts[0].y-P.y);
    if (std::abs(u.z)<1) return Vec3f(-1,1,1);
    return Vec3f(1.f-(u.x+u.y)/u.z, u.y/u.z, u.x/u.z); 
} 
 
void triangle(Vec2i *pts, TGAImage &image, TGAColor color) { 
    Vec2i bboxmin(image.get_width()-1,  image.get_height()-1); 
    Vec2i bboxmax(0, 0); 
    Vec2i clamp(image.get_width()-1, image.get_height()-1); 
    for (int i=0; i<3; i++) { 
        bboxmin.x = std::max(0, std::min(bboxmin.x, pts[i].x));
	bboxmin.y = std::max(0, std::min(bboxmin.y, pts[i].y));

	bboxmax.x = std::min(clamp.x, std::max(bboxmax.x, pts[i].x));
	bboxmax.y = std::min(clamp.y, std::max(bboxmax.y, pts[i].y));
    } 
    Vec2i P; 
    for (P.x=bboxmin.x; P.x<=bboxmax.x; P.x++) { 
        for (P.y=bboxmin.y; P.y<=bboxmax.y; P.y++) { 
            Vec3f bc_screen  = barycentric(pts, P); 
            if (bc_screen.x<0 || bc_screen.y<0 || bc_screen.z<0) continue; 
            image.set(P.x, P.y, color); 
        } 
    } 
} 
```

以上两节代码本人放到 [starthlw/LearningSoftRasterization at lesson2 (github.com)](https://github.com/starthlw/LearningSoftRasterization/tree/lesson2)

下一节代码发生了比较大的变动。

# 3 Hidden faces removal (z buffer)

深度判断无非就是在着色的时候增加一个相机方向的深度比较，更靠近相机则进行着色计算

```c

void triangle(Vec3i t0, Vec3i t1, Vec3i t2, TGAImage& image, TGAColor color, int* zbuffer) {
    if (t0.y == t1.y && t0.y == t2.y) return; 
    if (t0.y > t1.y) std::swap(t0, t1);
    if (t0.y > t2.y) std::swap(t0, t2);
    if (t1.y > t2.y) std::swap(t1, t2);
    int total_height = t2.y - t0.y;
    for (int i = 0; i < total_height; i++) {
        bool second_half = i > t1.y - t0.y || t1.y == t0.y;
        int segment_height = second_half ? t2.y - t1.y : t1.y - t0.y;
        float alpha = (float)i / total_height;
        float beta = (float)(i - (second_half ? t1.y - t0.y : 0)) / segment_height;
        Vec3i A = t0 + (t2 - t0) * alpha;
        Vec3i B = second_half ? t1 + (t2 - t1) * beta : t0 + (t1 - t0) * beta;
        if (A.x > B.x) std::swap(A, B);
        for (int j = A.x; j <= B.x; j++) {
            float phi = B.x == A.x ? 1. : (float)(j - A.x) / (float)(B.x - A.x);
            Vec3i P = A + (B - A) * phi;
            P.x = j; P.y = t0.y + i; // a hack to fill holes (due to int cast precision problems)
            int idx = j + (t0.y + i) * width;
            if (zbuffer[idx] < P.z) { // 此处增加深度判断
                zbuffer[idx] = P.z;
                image.set(P.x, P.y, color); // attention, due to int casts t0.y+i != A.y
            }
        }
    }
}
```



# 4 Perspective projection

GAMES101已有深入讲解，待补充推导计算



# 5 Moving the camera

要移动相机，则需要较为完整的渲染过程：

1. vertex shader中完成local->world->view->viewport的坐标转化
   1. 根据视角位置，up方向，世界坐标原点直接获取ModelView矩阵，即view * model
   2. 根据视角位置和世界坐标原点计算投影矩阵
   3. 根据屏幕大小计算viewport矩阵
   4. gl_Vertex = Viewport * Projection * ModelView * gl_Vertex; 
2. fragment shader中完成着色计算

大体代码如下：

```c
void viewport(int x, int y, int w, int h) {
    Viewport = Matrix::identity();
    Viewport[0][3] = x+w/2.f;
    Viewport[1][3] = y+h/2.f;
    Viewport[2][3] = 255.f/2.f;
    Viewport[0][0] = w/2.f;
    Viewport[1][1] = h/2.f;
    Viewport[2][2] = 255.f/2.f;
}

void projection(float coeff) {
    Projection = Matrix::identity();
    Projection[3][2] = coeff;
}

void lookat(Vec3f eye, Vec3f center, Vec3f up) {
    Vec3f z = (eye-center).normalize();
    Vec3f x = cross(up,z).normalize();
    Vec3f y = cross(z,x).normalize();
    ModelView = Matrix::identity();
    for (int i=0; i<3; i++) {
        ModelView[0][i] = x[i];
        ModelView[1][i] = y[i];
        ModelView[2][i] = z[i];
        ModelView[i][3] = -center[i];
    }
}

void triangle(Vec3i *pts, IShader &shader, TGAImage &image, TGAImage &zbuffer) {
    Vec2i bboxmin( std::numeric_limits<int>::max(),  std::numeric_limits<int>::max());
    Vec2i bboxmax(-std::numeric_limits<int>::max(), -std::numeric_limits<int>::max());
    for (int i=0; i<3; i++) {
        for (int j=0; j<2; j++) {
            bboxmin[j] = std::min(bboxmin[j], pts[i][j]);
            bboxmax[j] = std::max(bboxmax[j], pts[i][j]);
        }
    }
    Vec3i P;
    TGAColor color;
    for (P.x=bboxmin.x; P.x<=bboxmax.x; P.x++) {
        for (P.y=bboxmin.y; P.y<=bboxmax.y; P.y++) {
            Vec3f c = barycentric(pts[0], pts[1], pts[2], P);
            P.z = std::max(0, std::min(255, int(pts[0].z*c.x + pts[1].z*c.y + pts[2].z*c.z + .5))); // clamping to 0-255 since it is stored in unsigned char
            if (c.x<0 || c.y<0 || c.z<0 || zbuffer.get(P.x, P.y)[0]>P.z) continue;
            bool discard = shader.fragment(c, color);
            if (!discard) {
                zbuffer.set(P.x, P.y, TGAColor(P.z));
                image.set(P.x, P.y, color);
            }
        }
    }
}

int main(int argc, char** argv) {
    // ...
    lookat(eye, center, up);
    viewport(width/8, height/8, width*3/4, height*3/4);
    projection(-1.f/(eye-center).norm());
    light_dir.normalize();

    TGAImage image  (width, height, TGAImage::RGB);
    TGAImage zbuffer(width, height, TGAImage::GRAYSCALE);

    GouraudShader shader;
    for (int i=0; i<model->nfaces(); i++) {
        Vec3i screen_coords[3];
        for (int j=0; j<3; j++) {
            screen_coords[j] = shader.vertex(i, j);
        }
        triangle(screen_coords, shader, image, zbuffer);
    }

    image.  flip_vertically();
    zbuffer.flip_vertically();
    image.  write_tga_file("output.tga");
    zbuffer.write_tga_file("zbuffer.tga");

    delete model;
    return 0;
}
```























