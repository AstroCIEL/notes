# ip core

> /TCORE_0929/
> /TCORE_0929/TCORE_rtl/rtl_sim/list.f

```text
/../accum_buffer_rtl/psum_buffer.v
/../accum_buffer_rtl/rf_hde_128bx128.v
/../accum_buffer_rtl/psum_buffer_sc_ga.v
/../accum_buffer_rtl/data_scatter.v
/../accum_buffer_rtl/data_gather.v
/../controller_rtl/rtl/accu_ctrl.v
/../shift_accu_rtl/rtl/shift_accu.v
/../shift_accu_rtl/rtl/sub_shift_accu.v
/../shift_accu_rtl/rtl/multi_prec_add.v
/rtl/TCORE_accu.v
```

## 层次

```text
TCORE_accu(TCORE_accu.v)
    u_psum_buffer_sc_ga psum_buffer_sc_ga (psum_buffer_sc_ga.v)
        u_psum_buffer psum_buffer (psum_buffer.v)
            regfile rf_hde_128bx128 (rf_hde_128bx128.v)
        u_data_scatter data_scatter (data_scatter.v)
        u_data_gather data_gather (data_gather.v)
    u_accu_ctrl accu_ctrl (accu_ctrl.v)
    u_shift_accu shift_accu (shift_accu.v)
        u_sub_shift_accu sub_shift_accu (sub_shift_accu.v)(16)
            multi_prec_add1 multi_prec_add (multi_prec_add.v)
```

