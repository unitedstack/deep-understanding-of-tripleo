## TripleO 安装、配置、使用目前遇到的问题

1. 如何灵活的配置ceph？
   1. crushmap
   2. ssd与sata混合
   3. 使用对象存储
2. 如何配置计算节点HA
   1. 使用pacemaker remote
   2. 集成rock
3. 离线部署TripleO
4. 集成halo 



虚拟环境instack.json 
```
{
  "arch": "x86_64",
  "host-ip": "192.168.122.1",
  "power_manager": "nova.virt.baremetal.virtual_power_driver.VirtualPowerManager",
  "seed-ip": "",
  "ssh-key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEogIBAAKCAQEAtBI6DK5CFAxPJJEoLgnI5v/kn0ncV26o9wP9+di7czMhD6WX\ndfLtn2WNALVRopIVXDwb78JqPQEpXgWEZGIv4JIteYdh/GrdQhnmqEL/6FpMjMfZ\nnGPclfzg6dM2khRFexaf50G+bLb5kgIpFLOG0DJBI/r36lMVRz5I2LwKixWNeEIX\nz445SwPj4lUlbfjoodAPEX8HLQanCvaavTNDVvq5q8Qb3fQ2gXScA1crRUN9uMv0\n+JZTFbwkqQHepMb9DJKxHF6BH9tE5+Ttmc0Ra1eel0rteXK2A6CYX+vjiqtQkuNt\ntYtyKNvmKmhv4udd2YaqK/nGoKZEgULpcgfeUQIDAQABAoIBAGJzQKekMl5xqGeO\nsVASa3PYXi+0mzJ2PwzmctpB46KFRsMePuPu0HoAdIn5mEtw4RrPhlqciacW1n4g\nOBUGFbULVq+GFE2EQ7obHR/Lmcx4ajfiIBjABF9ApdtRbhmJ2b8FTKGMMUeQ9nwc\nkEdQLBnyD+lTEm5bxFtyMzPEA2OsliuV+R/7W892+JBAaNsvTUrW1+rm/9oxCoQv\nu/dxIAgiNnUULZvcCEBoZaiHQsdDyM1zUAkpoWBmp4kLICXC8xYHZUyFHp3ScHZV\nKKqlxbS/+61qUt7egCosb3GTP+KV5etd0MO2zN8TpNgkRPzZ7y9B/LmkHTlaazD9\nqSa1IgECgYEA61fxkRvtVB80sn53rw1XUT1RyWTYcHobDWCEmXPmtQERyvSndO4i\nC2540s1QPkZbZ3Fk2pvgm2+8NqTf0EFJDSan84HS0j7+x4tG4F6ijPTXL0DsClY2\nbrJIDTLAhWLulHKVd3HRNK4OaUSLazFIknIA+C8uZvsJkNLSWG4xBbECgYEAw+BX\nH20SVRJpY3rK3wrYIVcpDXDQm61nyJVGIO3BBH/GmviCb+E9xecahZSO+qq1wnad\nIJpp1lsdzU9iBy402TFL9nRMHb4JsR6Id7sNXV7rX3zaC3JGKqJiJEPKo+U1Mn1f\nZrBi73t/ylVKX5n8gOfeFslwmDquQJ0mlSRYaqECgYAL+AQEEjyGq7OdZEsn7vDC\n4/B14pgTWFJp4r+7oiZYjD5gaQLfMoEuvaaNaf2rvR5G64BqkcThgtQ6nzX2vGs/\nrPibrL2RDb0dXtry7D0uGAGdmJqoh+vqw0xgx3T9E6P4jr9FPNeb60I2XlMM14vO\nTtf3x0Z/3EKHSAGEl84McQKBgHDc2RZwgHmoTDVX0YFG/FXppOvrryekeQJokKn0\nlJ0FCujMfEv+2tsnWG7TtLbWmjhcpBjfIFC0260rKm68vxLOhtiRFjKlB2yZDUT/\n8Kl2QeUZSYIC7E8wlaATt7VMIqTe/JNs2vTmkjGBh4MidQ3JjHxQwaHVXgY5Brw0\n3wVBAoGAJsbIHlcKsX8q4hU/Sp2VwohZUmwR3eooTfVMmQjXdI0h3g8H/I0XzA5W\nqHcMJ/5ba4w6sztYRnGn8jIlyozhI9lGv/ajYPcbS3nuE7nEl98vbve8hcLP+VCJ\nkbMz+s1SELnexCmGQHdHxUp3nuERwd2xzQPBEYE6N+VlsATKrgg=\n-----END RSA PRIVATE KEY-----\n",
  "ssh-user": "root",
  "nodes": [
    {
      "mac": [
        "00:f0:09:b0:54:80"
      ],
      "cpu": "1",
      "memory": "6144",
      "disk": "40",
      "arch": "x86_64",
      "pm_user": "root",
      "pm_addr": "192.168.122.1",
      "pm_password": "-----BEGIN RSA PRIVATE KEY-----\nMIIEogIBAAKCAQEAtBI6DK5CFAxPJJEoLgnI5v/kn0ncV26o9wP9+di7czMhD6WX\ndfLtn2WNALVRopIVXDwb78JqPQEpXgWEZGIv4JIteYdh/GrdQhnmqEL/6FpMjMfZ\nnGPclfzg6dM2khRFexaf50G+bLb5kgIpFLOG0DJBI/r36lMVRz5I2LwKixWNeEIX\nz445SwPj4lUlbfjoodAPEX8HLQanCvaavTNDVvq5q8Qb3fQ2gXScA1crRUN9uMv0\n+JZTFbwkqQHepMb9DJKxHF6BH9tE5+Ttmc0Ra1eel0rteXK2A6CYX+vjiqtQkuNt\ntYtyKNvmKmhv4udd2YaqK/nGoKZEgULpcgfeUQIDAQABAoIBAGJzQKekMl5xqGeO\nsVASa3PYXi+0mzJ2PwzmctpB46KFRsMePuPu0HoAdIn5mEtw4RrPhlqciacW1n4g\nOBUGFbULVq+GFE2EQ7obHR/Lmcx4ajfiIBjABF9ApdtRbhmJ2b8FTKGMMUeQ9nwc\nkEdQLBnyD+lTEm5bxFtyMzPEA2OsliuV+R/7W892+JBAaNsvTUrW1+rm/9oxCoQv\nu/dxIAgiNnUULZvcCEBoZaiHQsdDyM1zUAkpoWBmp4kLICXC8xYHZUyFHp3ScHZV\nKKqlxbS/+61qUt7egCosb3GTP+KV5etd0MO2zN8TpNgkRPzZ7y9B/LmkHTlaazD9\nqSa1IgECgYEA61fxkRvtVB80sn53rw1XUT1RyWTYcHobDWCEmXPmtQERyvSndO4i\nC2540s1QPkZbZ3Fk2pvgm2+8NqTf0EFJDSan84HS0j7+x4tG4F6ijPTXL0DsClY2\nbrJIDTLAhWLulHKVd3HRNK4OaUSLazFIknIA+C8uZvsJkNLSWG4xBbECgYEAw+BX\nH20SVRJpY3rK3wrYIVcpDXDQm61nyJVGIO3BBH/GmviCb+E9xecahZSO+qq1wnad\nIJpp1lsdzU9iBy402TFL9nRMHb4JsR6Id7sNXV7rX3zaC3JGKqJiJEPKo+U1Mn1f\nZrBi73t/ylVKX5n8gOfeFslwmDquQJ0mlSRYaqECgYAL+AQEEjyGq7OdZEsn7vDC\n4/B14pgTWFJp4r+7oiZYjD5gaQLfMoEuvaaNaf2rvR5G64BqkcThgtQ6nzX2vGs/\nrPibrL2RDb0dXtry7D0uGAGdmJqoh+vqw0xgx3T9E6P4jr9FPNeb60I2XlMM14vO\nTtf3x0Z/3EKHSAGEl84McQKBgHDc2RZwgHmoTDVX0YFG/FXppOvrryekeQJokKn0\nlJ0FCujMfEv+2tsnWG7TtLbWmjhcpBjfIFC0260rKm68vxLOhtiRFjKlB2yZDUT/\n8Kl2QeUZSYIC7E8wlaATt7VMIqTe/JNs2vTmkjGBh4MidQ3JjHxQwaHVXgY5Brw0\n3wVBAoGAJsbIHlcKsX8q4hU/Sp2VwohZUmwR3eooTfVMmQjXdI0h3g8H/I0XzA5W\nqHcMJ/5ba4w6sztYRnGn8jIlyozhI9lGv/ajYPcbS3nuE7nEl98vbve8hcLP+VCJ\nkbMz+s1SELnexCmGQHdHxUp3nuERwd2xzQPBEYE6N+VlsATKrgg=\n-----END RSA PRIVATE KEY-----\n",
      "pm_type": "pxe_ssh"
    },
    {
      "mac": [
        "00:ad:d2:7d:84:3a"
      ],
      "cpu": "1",
      "memory": "6144",
      "disk": "40",
      "arch": "x86_64",
      "pm_user": "root",
      "pm_addr": "192.168.122.1",
      "pm_password": "-----BEGIN RSA PRIVATE KEY-----\nMIIEogIBAAKCAQEAtBI6DK5CFAxPJJEoLgnI5v/kn0ncV26o9wP9+di7czMhD6WX\ndfLtn2WNALVRopIVXDwb78JqPQEpXgWEZGIv4JIteYdh/GrdQhnmqEL/6FpMjMfZ\nnGPclfzg6dM2khRFexaf50G+bLb5kgIpFLOG0DJBI/r36lMVRz5I2LwKixWNeEIX\nz445SwPj4lUlbfjoodAPEX8HLQanCvaavTNDVvq5q8Qb3fQ2gXScA1crRUN9uMv0\n+JZTFbwkqQHepMb9DJKxHF6BH9tE5+Ttmc0Ra1eel0rteXK2A6CYX+vjiqtQkuNt\ntYtyKNvmKmhv4udd2YaqK/nGoKZEgULpcgfeUQIDAQABAoIBAGJzQKekMl5xqGeO\nsVASa3PYXi+0mzJ2PwzmctpB46KFRsMePuPu0HoAdIn5mEtw4RrPhlqciacW1n4g\nOBUGFbULVq+GFE2EQ7obHR/Lmcx4ajfiIBjABF9ApdtRbhmJ2b8FTKGMMUeQ9nwc\nkEdQLBnyD+lTEm5bxFtyMzPEA2OsliuV+R/7W892+JBAaNsvTUrW1+rm/9oxCoQv\nu/dxIAgiNnUULZvcCEBoZaiHQsdDyM1zUAkpoWBmp4kLICXC8xYHZUyFHp3ScHZV\nKKqlxbS/+61qUt7egCosb3GTP+KV5etd0MO2zN8TpNgkRPzZ7y9B/LmkHTlaazD9\nqSa1IgECgYEA61fxkRvtVB80sn53rw1XUT1RyWTYcHobDWCEmXPmtQERyvSndO4i\nC2540s1QPkZbZ3Fk2pvgm2+8NqTf0EFJDSan84HS0j7+x4tG4F6ijPTXL0DsClY2\nbrJIDTLAhWLulHKVd3HRNK4OaUSLazFIknIA+C8uZvsJkNLSWG4xBbECgYEAw+BX\nH20SVRJpY3rK3wrYIVcpDXDQm61nyJVGIO3BBH/GmviCb+E9xecahZSO+qq1wnad\nIJpp1lsdzU9iBy402TFL9nRMHb4JsR6Id7sNXV7rX3zaC3JGKqJiJEPKo+U1Mn1f\nZrBi73t/ylVKX5n8gOfeFslwmDquQJ0mlSRYaqECgYAL+AQEEjyGq7OdZEsn7vDC\n4/B14pgTWFJp4r+7oiZYjD5gaQLfMoEuvaaNaf2rvR5G64BqkcThgtQ6nzX2vGs/\nrPibrL2RDb0dXtry7D0uGAGdmJqoh+vqw0xgx3T9E6P4jr9FPNeb60I2XlMM14vO\nTtf3x0Z/3EKHSAGEl84McQKBgHDc2RZwgHmoTDVX0YFG/FXppOvrryekeQJokKn0\nlJ0FCujMfEv+2tsnWG7TtLbWmjhcpBjfIFC0260rKm68vxLOhtiRFjKlB2yZDUT/\n8Kl2QeUZSYIC7E8wlaATt7VMIqTe/JNs2vTmkjGBh4MidQ3JjHxQwaHVXgY5Brw0\n3wVBAoGAJsbIHlcKsX8q4hU/Sp2VwohZUmwR3eooTfVMmQjXdI0h3g8H/I0XzA5W\nqHcMJ/5ba4w6sztYRnGn8jIlyozhI9lGv/ajYPcbS3nuE7nEl98vbve8hcLP+VCJ\nkbMz+s1SELnexCmGQHdHxUp3nuERwd2xzQPBEYE6N+VlsATKrgg=\n-----END RSA PRIVATE KEY-----\n",
      "pm_type": "pxe_ssh"
    }
  ]
}

```