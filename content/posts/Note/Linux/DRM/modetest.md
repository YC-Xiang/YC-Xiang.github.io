```shell
# modetest --help
usage: modetest [-acDdefMPpsCvrw]

 Query options:
        -c      list connectors
        -e      list encoders
        -f      list framebuffers
        -p      list CRTCs and planes (pipes)

 Test options:
        -P <plane_id>@<crtc_id>:<w>x<h>[+<x>+<y>][*<scale>][@<format>]  set a plane
        -s <connector_id>[,<connector_id>][@<crtc_id>]:[#<mode index>]<mode>[-<vrefresh>][@<format>]    set a mode
        -C      test hw cursor
        -v      test vsynced page flipping
        -r      set the preferred mode for all connectors
        -w <obj_id>:<prop_name>:<value> set property
        -a      use atomic API
        -F pattern1,pattern2    specify fill patterns

 Generic options:
        -d      drop master after mode set
        -M module       use the given driver
        -D device       use the given device
```

```shell
# dump all info
modetest -M rts_drm

# 设置 preferred mode
modetest -M rts_drm -a -r

# test vsync
modetest -M rts_drm -a -s 51@47:720x480i -P 31@47:720x480 -F tiles -v

# test double plane
modetest -M rts_drm -a -s 51@47:720x480i -P 31@47:720x480 -P 38@47:100x100+50+50@NV12 -F plain,smpte

# 设置 property
modetest -M rts_drm 47:BACKGROUND:256 # 加上 -a 选项会报错，看起来 modetest code 中的 dev->req 没设置
```
