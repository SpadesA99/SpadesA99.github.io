---
layout: post
title: antd pro权限控制
tags:
  - antd umijs
categories:
  - 前端
date: 2022-06-13 18:23:03
---



### antd pro权限控制方案

##### 参考路由定义

routes.ts 文件添加路由

```tsx
  {
    name: 'pageA',
    path: '/pageA',
    component: './Render',
    roles: ['admin', 'user'], //定义角色访问权限
    access: "auth", //使用 access auth 函数控制菜单访问权限
    wrappers: ['@/wrappers/auth'] //高级包装器控制页面访问权限
  },
  {
    name: 'pageB',
    path: '/pageB',
    component: './Render',
    roles: ['admin'],
    access: "auth",
    wrappers: ['@/wrappers/auth']
  },
  {
    name: 'pageC',
    path: '/pageC',
    component: './Render',
    roles: ['user'],
    access: "auth",
    wrappers: ['@/wrappers/auth']
  },
```

------

##### access

https://umijs.org/zh-CN/plugins/plugin-access

src/access.ts

```tsx
let hasAuth = (route: any, roleId?: string) => {
  return route.roles ? route.roles.includes(roleId) : true;
};

export default function access(initialState: { currentUser?: API.CurrentUser } | undefined) {
  const { currentUser } = initialState ?? {};
  return {
    auth: (route: any) => hasAuth(route, currentUser?.access),
  };
}
```

access 将返回一个anth方法进行规则匹配



------

##### wrappers

https://umijs.org/zh-CN/docs/routing#wrappers 

https://pro.ant.design/docs/authority-management#3-permission-control-for-routing-and-menus

src/wrappers/auth.tsx

```tsx	
import { useAccess } from 'umi'
import { useModel } from 'umi';

export default (props: any) => {
    const { initialState } = useModel('@@initialState');
    const access = useAccess();
    const { currentUser } = initialState || {};

    if (access.auth(props.route) && currentUser) {
        return <div>{props.children}</div>;
    } else {
        return <div>无权限</div>;
    }
}
```

