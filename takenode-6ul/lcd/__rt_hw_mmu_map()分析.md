

```c
//映射页面
static int __rt_hw_mmu_map(rt_mmu_info *mmu_info, void* v_addr, void* p_addr, size_t npages, size_t attr)
{
    size_t loop_va = (size_t)v_addr & ~ARCH_PAGE_MASK;//取出页号
    size_t loop_pa = (size_t)p_addr & ~ARCH_PAGE_MASK;
    size_t l1_off, l2_off;
    size_t *mmu_l1, *mmu_l2;
#ifndef RT_USING_USERSPACE
    size_t *ref_cnt;
#endif

    if (!mmu_info)
    {
        return -1;
    }

    while (npages--)
    {
        l1_off = (loop_va >> ARCH_SECTION_SHIFT);
        l2_off = ((loop_va & ARCH_SECTION_MASK) >> ARCH_PAGE_SHIFT);
        mmu_l1 =  (size_t*)mmu_info->vtable + l1_off;//做映射
		//先判断一级项有没有映射到二级页表
        if (*mmu_l1 & ARCH_MMU_USED_MASK)
        {
        	//直接把二级页表取出来用
            mmu_l2 = (size_t *)((*mmu_l1 & ~ARCH_PAGE_TBL_MASK) - mmu_info->pv_off);//做映射
            rt_page_ref_inc(mmu_l2, 0);//统计页面加引用数
        }
				//没有就创建一个二级页表
        else
        {
#ifdef RT_USING_USERSPACE
            mmu_l2 = (size_t*)rt_pages_alloc(0);//分配内存页
#else
            mmu_l2 = (size_t*)rt_malloc_align(ARCH_PAGE_TBL_SIZE * 2, ARCH_PAGE_TBL_SIZE);//做内存对齐
#endif
            if (mmu_l2)
            {
                rt_memset(mmu_l2, 0, ARCH_PAGE_TBL_SIZE * 2);
                /* cache maintain */
                rt_hw_cpu_dcache_clean(mmu_l2, ARCH_PAGE_TBL_SIZE);//页表操作从cache同步到ram

                *mmu_l1 = (((size_t)mmu_l2 + mmu_info->pv_off) | 0x1);
                /* cache maintain */
                rt_hw_cpu_dcache_clean(mmu_l1, 4);
            }
            else
            {
                /* error, unmap and quit */
                __rt_hw_mmu_unmap(mmu_info, v_addr, npages);
                return -1;
            }
        }

#ifndef RT_USING_USERSPACE
        ref_cnt = mmu_l2 + (ARCH_SECTION_SIZE / ARCH_PAGE_SIZE);
        (*ref_cnt)++;
#endif

        *(mmu_l2 + l2_off) = (loop_pa | attr);//设置页面属性MMU_MAP_U_RWCB
        /* cache maintain */
        rt_hw_cpu_dcache_clean(mmu_l2 + l2_off, 4);

        loop_va += ARCH_PAGE_SIZE;
        loop_pa += ARCH_PAGE_SIZE;
    }
    return 0;
}
```
  