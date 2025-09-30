---
title: "直播流缓存的一些事"
date: 2025-10-01
draft: true
tags: ["HLS", "Stream", "Cache"]
---

```bash
ffmpeg -i "https://cn-hnld-ct-01-21.bilivideo.com/live-bvc/536553/live_2124647716_1414766_bluray/index.m3u8?expires=1759252762&len=0&oi=2085476355&pt=h5&qn=10000&trid=1007606c082fc14ab58648aff49d6268dc03&bmt=1&sigparams=cdn,expires,len,oi,pt,qn,trid,bmt&cdn=cn-gotcha01&sign=2416df1332c509e8649435d9512e4465&site=023962dd910109d4c54da396f31556f0&free_type=0&mid=475210&sche=ban&bvchls=1&sid=cn-hnld-ct-01-21&chash=0&sg=lr&trace=8388633&isp=ct&rg=East&pv=Shanghai&deploy_env=prod&score=42&p2p_type=-1&sl=2&strategy_types=1&codec=0&strategy_version=latest&hdr_type=0&source=puv3_onetier&long_ab_id=45&strategy_ids=11&hot_cdn=57345&suffix=bluray&origin_bitrate=8128&long_ab_flag=live_default_longitudinal&sk=11e6e03140d4fe707653c968459991d32b386f4c30439c091581a0d0389f9eb4&media_type=0&long_ab_flag_value=test&pp=rtmp&info_source=origin&flvsk=7b9ca197a822eebb7933bf2a49b49d0f2b5784eba95887cea6a1ec4d52e303b6&vd=nc&zoneid_l=151355393&sid_l=live_2124647716_1414766_bluray&src=puv3&order=1" \
 -hls*list_size 0 \
 -hls_time 5 \
 -hls_flags append_list+program_date_time \
 -hls_segment_filename "cache*%Y%m%d*%H%M%S*%03d.ts" \
 -strftime 1 \
 -c copy \
 playlist.m3u8
```
