## 想法及框架

用django框架来创建一个照片功能的网站。增加照片页面 

-  新增一个插入照片功能，将照片放置到腾讯云图床中 -
- 建立一个MySQL数据库，放置照片分类、照片拍摄时间、照片创建时间、照片的图床url地址、照片标题、照片描述等信息  -
- 可以以时间顺序的方式来网页上展示照片



实现步骤：

---

### **1. 创建项目和应用**
1. **创建 Django 项目和应用**
   ```bash
   django-admin startproject photowebsite
   cd photowebsite
   python manage.py startapp photoapp
   ```

2. **注册应用**
   在 `settings.py` 的 `INSTALLED_APPS` 中加入 `photoapp`：
   ```python
   INSTALLED_APPS = [
       ...
       'photoapp',
   ]
   ```

---

### **2. 配置 MySQL 数据库**
1. 安装 MySQL 相关驱动：
   ```bash
   pip install mysqlclient
   ```

2. 修改 `settings.py`，设置 MySQL 数据库：
   ```python
   DATABASES = {
       'default': {
           'ENGINE': 'django.db.backends.mysql',
           'NAME': 'photo_db',  # 数据库名称
           'USER': 'root',      # 数据库用户名
           'PASSWORD': 'yourpassword',  # 数据库密码
           'HOST': 'localhost', # 数据库地址
           'PORT': '3306',      # 数据库端口
       }
   }
   ```

3. **创建模型**
   在 `photoapp/models.py` 中定义照片表：
   ```python
   from django.db import models
   
   class Photo(models.Model):
       title = models.CharField(max_length=255)  # 照片标题
       description = models.TextField(blank=True)  # 照片描述
       category = models.CharField(max_length=100)  # 分类
       url = models.URLField()  # 图床 URL
       taken_date = models.DateField()  # 拍摄时间
       created_at = models.DateTimeField(auto_now_add=True)  # 创建时间
   
       def __str__(self):
           return self.title
   ```

4. **迁移数据库**
   ```bash
   python manage.py makemigrations
   python manage.py migrate
   ```

---

### **3. 配置腾讯云图床**
1. 安装 `qcloud-cos` SDK：
   ```bash
   pip install cos-python-sdk-v5
   ```

2. 在项目根目录下创建 `utils/qcloud_cos.py`，封装上传逻辑：
   ```python
   from qcloud_cos import CosConfig, CosS3Client
   import sys
   import logging
   
   def upload_to_tencent(file, file_name):
       secret_id = '你的SecretId'
       secret_key = '你的SecretKey'
       region = 'ap-region'  # 替换为你的存储桶区域，例如 'ap-shanghai'
       bucket = 'your-bucket-name'
   
       config = CosConfig(Region=region, SecretId=secret_id, SecretKey=secret_key)
       client = CosS3Client(config)
   
       try:
           response = client.upload_file_from_buffer(
               Bucket=bucket,
               Body=file,
               Key=file_name
           )
           return f"https://{bucket}.cos.{region}.myqcloud.com/{file_name}"
       except Exception as e:
           logging.error(f"Failed to upload file: {e}")
           raise e
   ```

---

### **4. 添加插入照片的视图和模板**
1. **视图函数**
   在 `photoapp/views.py` 中新增：
   ```python
   from django.shortcuts import render, redirect
   from .models import Photo
   from .forms import PhotoForm
   from utils.qcloud_cos import upload_to_tencent
   
   def add_photo(request):
       if request.method == 'POST':
           form = PhotoForm(request.POST, request.FILES)
           if form.is_valid():
               photo = form.save(commit=False)
               file = request.FILES['photo_file']
               file_name = f"photos/{file.name}"
               photo.url = upload_to_tencent(file, file_name)
               photo.save()
               return redirect('photo_list')
       else:
           form = PhotoForm()
       return render(request, 'photoapp/add_photo.html', {'form': form})
   
   def photo_list(request):
       photos = Photo.objects.all().order_by('-taken_date')
       return render(request, 'photoapp/photo_list.html', {'photos': photos})
   ```

2. **表单**
   在 `photoapp/forms.py` 中创建表单：
   ```python
   from django import forms
   from .models import Photo
   
   class PhotoForm(forms.ModelForm):
       photo_file = forms.ImageField(label="上传图片", required=True)
   
       class Meta:
           model = Photo
           fields = ['title', 'description', 'category', 'taken_date']
   ```

3. **模板**
   - **`add_photo.html`**
     ```html
     <h1>添加照片</h1>
     <form method="post" enctype="multipart/form-data">
         {% csrf_token %}
         {{ form.as_p }}
         <button type="submit">上传</button>
     </form>
     ```

   - **`photo_list.html`**
     ```html
     <h1>照片列表</h1>
     <ul>
         {% for photo in photos %}
             <li>
                 <h3>{{ photo.title }}</h3>
                 <img src="{{ photo.url }}" alt="{{ photo.title }}" width="200">
                 <p>{{ photo.description }}</p>
                 <p>拍摄时间: {{ photo.taken_date }}</p>
                 <p>创建时间: {{ photo.created_at }}</p>
             </li>
         {% endfor %}
     </ul>
     ```

---

### **5. 配置路由**
1. 在 `photoapp/urls.py` 中：
   ```python
   from django.urls import path
   from . import views
   
   urlpatterns = [
       path('add/', views.add_photo, name='add_photo'),
       path('', views.photo_list, name='photo_list'),
   ]
   ```

2. 在项目 `urls.py` 中引入：
   ```python
   from django.urls import path, include
   
   urlpatterns = [
       path('photos/', include('photoapp.urls')),
   ]
   ```

---

### **6. 配置静态文件和媒体文件**
确保 `settings.py` 中配置了静态文件和媒体文件的支持：
```python
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

STATIC_URL = '/static/'
STATICFILES_DIRS = [BASE_DIR / 'static']
```

---

### **7. 测试运行**
1. 启动开发服务器：
   ```bash
   python manage.py runserver
   ```

2. 访问 `http://127.0.0.1:8000/photos/` 查看照片列表。
3. 访问 `http://127.0.0.1:8000/photos/add/` 添加照片。

---

