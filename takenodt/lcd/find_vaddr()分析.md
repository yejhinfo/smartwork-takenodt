`

```c
static size_t find_vaddr(rt_mmu_info *mmu_info, int pages)
{
    size_t l1_off, l2_off;
    size_t *mmu_l1, *mmu_l2;
    size_t find_off = 0;
    size_t find_va = 0;
    int n = 0;
if (!pages)
{
    return 0;
}

if (!mmu_info)
{
    return 0;
}

for (l1_off = mmu_info->vstart; l1_off <= mmu_info->vend; l1_off++)
{
    mmu_l1 =  (size_t*)mmu_info->vtable + l1_off;//当前页表的虚拟地址的起始位置
    if (*mmu_l1 & ARCH_MMU_USED_MASK)//如果从这个位置开始的内存有人在使用
    {
        mmu_l2 = (size_t *)((*mmu_l1 & ~ARCH_PAGE_TBL_MASK) - mmu_info->pv_off);//减掉 就是把物理地址转换成虚拟地址，得从新找个表，之前得表有人在使用了
        for (l2_off = 0; l2_off < (ARCH_SECTION_SIZE/ARCH_PAGE_SIZE); l2_off++)
        {
            if (*(mmu_l2 + l2_off) & ARCH_MMU_USED_MASK)//不断的找没人使用的内存，直到找到没人用的
            {
                /* in use */
                n = 0;
            }
            else
            {
                if (!n)
                {
                    find_va = l1_off;
                    find_off = l2_off;
                }
                n++;
                if (n >= pages)
                {
                    return (find_va << ARCH_SECTION_SHIFT) + (find_off << ARCH_PAGE_SHIFT);
                }
            }
        }
    }
    else
    {
        if (!n)//这个位置开始的内存没人在使用
        {
            find_va = l1_off;
            find_off = 0;
        }
        n += (ARCH_SECTION_SIZE/ARCH_PAGE_SIZE);//不断的在一偏连续空间进行内存技术
        if (n >= pages)//分配的内存满足pages时，内存分配完毕
        {
            return (find_va << ARCH_SECTION_SHIFT) + (find_off << ARCH_PAGE_SHIFT);//将这片内存返回出去
        }
    }
}
return 0;
```
}`