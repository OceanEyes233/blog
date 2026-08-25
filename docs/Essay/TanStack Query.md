# TanStack Query简介

TanStack Query 是一个专门解决服务端状态管理的库，它可以帮助我们管理客户端组件的远程数据状态，包括数据获取、缓存、失效、重拉等。

```
"use client"

import {
  QueryClient,
  QueryClientProvider,
} from '@tanstack/react-query'
import { useState } from 'react'

// QueryClient 必须放在 client provider 中创建，避免服务端组件环境下共享同一个运行时实例。
export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            retry: 1,
            staleTime: 30_000,
          },
        },
      }),
  )

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}