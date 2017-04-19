## undercloud镜像

###undercloud镜像的位置undercloud_image_url这个参数来指定
```
undercloud_image_url="url"
```

###使用undercloud image cache
部署时指定以下4个参数，可以使用cached image。
```
force_cached_images=true
image_cache_dir=<location on undercloud>
image.name=<image name>
image.type=<image type>

#镜像需要放在 image_cache_dir 目录中，并以 latest-{{ image.name }}.{{ image.type }} 格式命名
# 例如/var/cache/tripleo-quickstart/images/latest-undercloud.qcow2
{{ image_cache_dir }}/latest-{{ image.name }}.{{ image.type }}
```