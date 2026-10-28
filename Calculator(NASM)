; ============================================================
;  Инженерный калькулятор
;  x86-64, Linux, NASM, x87
;  nasm -f elf64 calc.asm -o calc.o && ld calc.o -o calc
; ============================================================
global _start

BUFSZ       equ 4096            ; буфер ввода (макс. длина строки BUFSZ-1)
VSTACK_SZ   equ 256             ; глубина стека значений (qword)
RC_DOWN     equ 0x0400          ; режимы округления x87
RC_UP       equ 0x0800

section .data
    ten        dq 10.0
    f_half     dq 0.5
    f_two      dq 2.0
    f_three    dq 3.0
    f_23       dq 23.0
    f_90       dq 90.0
    f_180      dq 180.0
    f_360      dq 360.0
    f_1800     dq 1800.0
    f_5000     dq 5000.0
    f_1e11     dq 1.0e11
    f_1e_15    dq 1.0e-15
    f_l1p      dq 0.29                   ; граница для fyl2xp1
    f_2p63     dq 0x43E0000000000000     ; 2^63 — предел fsin/fcos/fptan
    f_big      dq 32768.0                ; |t| выше — 2^t точно inf/0
    f_inf      dq 0x7FF0000000000000
    f_nan      dq 0x7FF8000000000000

    prompt_rad db "rad> "
    prompt_deg db "deg> "
    prompt_len equ 5

    help_msg   db "=== Инженерный калькулятор (x87) ===", 10
               db "Операторы:  + - * /  % остаток  ^ степень  ! факториал", 10
               db "Приоритет:  !  >  ^  >  унарный -  >  * / %  >  + -", 10
               db "Скобки:     (1+2)*3   2(3+4)   (1+2)(3+4)   2pi   3x", 10
               db "Числа:      12  .5  3.14  1.5e-3", 10
               db "Константы:  pi  e  ans (прошлый результат)", 10
               db "Переменные: a..z (кроме e):  x = 5,  потом 2x+1", 10
               db "Тригоном.:  sin cos tan asin acos atan atan2(y,x)", 10
               db "Гиперб.:    sinh cosh tanh asinh acosh atanh", 10
               db "Прочие:     sqrt cbrt exp ln log log2 abs floor ceil round", 10
               db "2 аргумента: min(a,b) max(a,b) root(x,n) ncr(n,k) npr(n,k)", 10
               db "            sin(x)^2 = (sin x)^2", 10
               db "Команды:    deg  rad  help  q", 10, 10
    help_len   equ $ - help_msg

    err_pre    db "ошибка: позиция "
    err_pre_len equ $ - err_pre
    long_msg   db "ошибка: строка слишком длинная", 10
    long_len   equ $ - long_msg
    msg_deg    db "режим: градусы", 10
    msg_deg_len equ $ - msg_deg
    msg_rad    db "режим: радианы", 10
    msg_rad_len equ $ - msg_rad

    ; команды
    c_q     db "q",0
    c_quit  db "quit",0
    c_exit  db "exit",0
    c_deg   db "deg",0
    c_rad   db "rad",0
    c_help  db "help",0
    ; константы
    k_pi    db "pi",0
    k_e     db "e",0
    k_ans   db "ans",0
    ; функции
    n_sin   db "sin",0
    n_cos   db "cos",0
    n_tan   db "tan",0
    n_asin  db "asin",0
    n_acos  db "acos",0
    n_atan  db "atan",0
    n_sinh  db "sinh",0
    n_cosh  db "cosh",0
    n_tanh  db "tanh",0
    n_asinh db "asinh",0
    n_acosh db "acosh",0
    n_atanh db "atanh",0
    n_sqrt  db "sqrt",0
    n_cbrt  db "cbrt",0
    n_exp   db "exp",0
    n_ln    db "ln",0
    n_log   db "log",0
    n_log2  db "log2",0
    n_abs   db "abs",0
    n_floor db "floor",0
    n_ceil  db "ceil",0
    n_round db "round",0
    n_atan2 db "atan2",0
    n_min   db "min",0
    n_max   db "max",0
    n_root  db "root",0
    n_ncr   db "ncr",0
    n_npr   db "npr",0

    ; имя, обработчик, число аргументов
    align 8
functab:
    dq n_sin,   op_sin,   1
    dq n_cos,   op_cos,   1
    dq n_tan,   op_tan,   1
    dq n_asin,  op_asin,  1
    dq n_acos,  op_acos,  1
    dq n_atan,  op_atan,  1
    dq n_sinh,  op_sinh,  1
    dq n_cosh,  op_cosh,  1
    dq n_tanh,  op_tanh,  1
    dq n_asinh, op_asinh, 1
    dq n_acosh, op_acosh, 1
    dq n_atanh, op_atanh, 1
    dq n_sqrt,  op_sqrt,  1
    dq n_cbrt,  op_cbrt,  1
    dq n_exp,   op_exp,   1
    dq n_ln,    op_ln,    1
    dq n_log,   op_log,   1
    dq n_log2,  op_log2,  1
    dq n_abs,   op_abs,   1
    dq n_floor, op_floor, 1
    dq n_ceil,  op_ceil,  1
    dq n_round, op_round, 1
    dq n_atan2, op_atan2, 2
    dq n_min,   op_min,   2
    dq n_max,   op_max,   2
    dq n_root,  op_root,  2
    dq n_ncr,   op_ncr,   2
    dq n_npr,   op_npr,   2
    dq 0

    angle_mode db 0                      ; 0 = радианы, 1 = градусы
    align 8
    ans        dq 0.0
    var_def    dd 0                      ; битовая маска заданных a..z

section .bss
    inbuf     resb BUFSZ + 8
    outbuf    resb 128
    intbuf    resb 64
    dbuf      resb 16
    buf_start resq 1
    tmpq      resq 1
    rt_x      resq 1
    rt_n      resq 1
    pow_base  resq 1
    pow_exp   resq 1
    pow_exp_i resq 1
    vars      resq 26
    vstack    resq VSTACK_SZ

section .text
; ============================================================
;  _start: построчное чтение. rbx = байт в буфере,
;          r12 = 1 — выбрасываем хвост слишком длинной строки
; ============================================================
_start:
    mov eax, 1
    mov edi, 1
    lea rsi, [help_msg]
    mov edx, help_len
    syscall

    xor ebx, ebx
    xor r12d, r12d

main_loop:
    test rbx, rbx
    jnz .read
    lea rsi, [prompt_rad]
    cmp byte [angle_mode], 0
    je .pr
    lea rsi, [prompt_deg]
.pr:
    mov eax, 1
    mov edi, 1
    mov edx, prompt_len
    syscall

.read:
    xor eax, eax
    xor edi, edi
    lea rsi, [inbuf + rbx]
    mov edx, BUFSZ - 1
    sub rdx, rbx
    syscall
    test rax, rax
    jle .eof
    add rbx, rax

    test r12d, r12d
    jz .find
    xor ecx, ecx                    ; режим выброса: ищем первый '\n'
.dscan:
    cmp rcx, rbx
    jae .dall
    cmp byte [inbuf + rcx], 10
    je .dfound
    inc rcx
    jmp .dscan
.dall:
    xor ebx, ebx
    jmp .read
.dfound:
    inc rcx
    mov r15, rcx
    call shift_buf
    xor r12d, r12d

.find:                              ; последний '\n'
    mov rcx, rbx
.fscan:
    test rcx, rcx
    jz .nonl
    dec rcx
    cmp byte [inbuf + rcx], 10
    jne .fscan

    lea r15, [rcx + 1]
    movzx eax, byte [inbuf + r15]
    push rax
    mov byte [inbuf + r15], 0
    lea rsi, [inbuf]
    call process
    pop rax
    mov [inbuf + r15], al
    call shift_buf
    jmp main_loop

.nonl:
    cmp rbx, BUFSZ - 1
    jb .read
    mov eax, 1
    mov edi, 1
    lea rsi, [long_msg]
    mov edx, long_len
    syscall
    xor ebx, ebx
    mov r12d, 1
    jmp .read

.eof:
    test r12d, r12d
    jnz .quit
    test rbx, rbx
    jz .quit
    mov byte [inbuf + rbx], 0
    lea rsi, [inbuf]
    call process
.quit:
    mov eax, 60
    xor edi, edi
    syscall

; выкинуть первые r15 байт буфера (rbx -= r15)
shift_buf:
    lea rsi, [inbuf + r15]
    lea rdi, [inbuf]
    mov rcx, rbx
    sub rcx, r15
    sub rbx, r15
    cld
    rep movsb
    ret

; ============================================================
;  process: все строки в буфере (NUL-terminated)
; ============================================================
process:
    push r12
    push r13
    push r14
    push rbp
    lea rbp, [vstack]
    mov r13, rsi
    mov [buf_start], rsi

    mov rdi, rsi                    ; всё в нижний регистр
.lc:
    movzx eax, byte [rdi]
    test al, al
    jz .loop
    cmp al, 'A'
    jb .lc_n
    cmp al, 'Z'
    ja .lc_n
    add byte [rdi], 32
.lc_n:
    inc rdi
    jmp .lc

.loop:
    call skip_ws_nl
    movzx eax, byte [r13]
    test al, al
    jz .done

    lea rdi, [c_q]
    call match_cmd
    jnc .quit
    lea rdi, [c_quit]
    call match_cmd
    jnc .quit
    lea rdi, [c_exit]
    call match_cmd
    jnc .quit
    lea rdi, [c_deg]
    call match_cmd
    jnc .set_deg
    lea rdi, [c_rad]
    call match_cmd
    jnc .set_rad
    lea rdi, [c_help]
    call match_cmd
    jnc .help

    ; присваивание: буква [пробелы] '=' expr
    movzx eax, byte [r13]
    cmp al, 'a'
    jb .parse
    cmp al, 'z'
    ja .parse
    movzx edx, byte [r13 + 1]
    call is_alnum
    jnc .parse
    lea rsi, [r13 + 1]
.asp:
    cmp byte [rsi], ' '
    je .asp_n
    cmp byte [rsi], 9
    jne .asp_d
.asp_n:
    inc rsi
    jmp .asp
.asp_d:
    cmp byte [rsi], '='
    jne .parse
    cmp al, 'e'                     ; e — константа
    je .error
    sub eax, 'a'
    mov r12d, eax
    lea r13, [rsi + 1]
    xor r14d, r14d
    call expr
    jc .error
    call check_eol
    jc .error
    fst qword [ans]
    fst qword [vars + r12*8]
    bts dword [var_def], r12d
    lea rdi, [outbuf]
    lea eax, [r12 + 'a']
    mov [rdi], al
    mov dword [rdi + 1], ' = '
    add rdi, 4
    call print_num_at
    jmp .next

.parse:
    xor r14d, r14d
    call expr
    jc .error
    call check_eol
    jc .error
    fst qword [ans]
    call print_num
    jmp .next

.set_deg:
    mov byte [angle_mode], 1
    lea rsi, [msg_deg]
    mov edx, msg_deg_len
    jmp .say
.set_rad:
    mov byte [angle_mode], 0
    lea rsi, [msg_rad]
    mov edx, msg_rad_len
    jmp .say
.help:
    lea rsi, [help_msg]
    mov edx, help_len
.say:
    mov eax, 1
    mov edi, 1
    syscall

.next:
    movzx eax, byte [r13]
    test al, al
    jz .done
    inc r13                         ; '\n'
    jmp .loop

.quit:
    mov eax, 60
    xor edi, edi
    syscall

.error:
    fninit
    xor r14d, r14d
    mov rsi, r13                    ; начало строки
.bk:
    cmp rsi, [buf_start]
    jbe .bk_done
    cmp byte [rsi - 1], 10
    je .bk_done
    dec rsi
    jmp .bk
.bk_done:
    mov rax, r13
    sub rax, rsi
    inc rax                         ; позиция с 1
    lea rdi, [outbuf]
    lea rsi, [err_pre]
    mov ecx, err_pre_len
    rep movsb
    call put_u64
    mov byte [rdi], 10
    inc rdi
    lea rsi, [outbuf]
    mov rdx, rdi
    sub rdx, rsi
    mov eax, 1
    mov edi, 1
    syscall
.err_skip:
    movzx eax, byte [r13]
    test al, al
    jz .done
    inc r13
    cmp al, 10
    jne .err_skip
    jmp .loop

.done:
    pop rbp
    pop r14
    pop r13
    pop r12
    ret

; CF=1, если после выражения что-то кроме конца строки
check_eol:
    call skip_ws
    movzx eax, byte [r13]
    test al, al
    jz .ok
    cmp al, 10
    je .ok
    stc
    ret
.ok:
    clc
    ret

; match_cmd: слово rdi + конец строки. CF=0 — совпало.
match_cmd:
    push r13
    call match_str
    jc .no
    call check_eol
    jc .no
    add rsp, 8
    clc
    ret
.no:
    pop r13
    stc
    ret

; is_alnum: dl — буква a-z или цифра? CF=0 — да
is_alnum:
    cmp dl, '0'
    jb .no
    cmp dl, '9'
    jbe .yes
    cmp dl, 'a'
    jb .no
    cmp dl, 'z'
    ja .no
.yes:
    clc
    ret
.no:
    stc
    ret

; ------------------------------------------------------------
skip_ws:
    movzx eax, byte [r13]
    cmp al, ' '
    je .adv
    cmp al, 9
    je .adv
    cmp al, 13
    je .adv
    ret
.adv:
    inc r13
    jmp skip_ws

skip_ws_nl:
    movzx eax, byte [r13]
    cmp al, ' '
    je .adv
    cmp al, 9
    je .adv
    cmp al, 13
    je .adv
    cmp al, 10
    je .adv
    ret
.adv:
    inc r13
    jmp skip_ws_nl

; vpush: CF=1 при переполнении
vpush:
    cmp r14, VSTACK_SZ * 8
    jae .ovf
    fstp qword [rbp + r14]
    add r14, 8
    clc
    ret
.ovf:
    stc
    ret

vpop:
    sub r14, 8
    fld qword [rbp + r14]
    ret

; ============================================================
;  Парсер
;  expr    := term (('+'|'-') term)*
;  term    := unary (('*'|'/'|'%'|<неявное *>) unary)*
;  unary   := ('+'|'-')* power
;  power   := postfix ('^' unary)?
;  postfix := primary '!'*
;  primary := число | '(' expr ')' | функция | константа | переменная
; ============================================================
expr:
    push r12
    call term
    jc .fail
.loop:
    call skip_ws
    movzx eax, byte [r13]
    cmp al, '+'
    je .add
    cmp al, '-'
    je .sub
    clc
    pop r12
    ret
.add:
    xor r12d, r12d
    jmp .op
.sub:
    mov r12d, 1
.op:
    inc r13
    call vpush
    jc .fail
    call term
    jc .fail
    call vpop                ; st0=accum, st1=term
    test r12d, r12d
    jz .do_add
    fsubrp st1, st0          ; accum - term
    jmp .loop
.do_add:
    faddp st1, st0
    jmp .loop
.fail:
    stc
    pop r12
    ret

term:
    push r12
    call unary
    jc .fail
.loop:
    call skip_ws
    movzx eax, byte [r13]
    xor r12d, r12d
    cmp al, '*'
    je .op
    mov r12d, 1
    cmp al, '/'
    je .op
    mov r12d, 2
    cmp al, '%'
    je .op
    ; неявное умножение: 2(3)  (1)(2)  2pi  3x
    xor r12d, r12d
    cmp al, '('
    je .op_noinc
    cmp al, 'a'
    jb .end
    cmp al, 'z'
    jbe .op_noinc
.end:
    clc
    pop r12
    ret
.op:
    inc r13
.op_noinc:
    call vpush
    jc .fail
    call unary
    jc .fail
    call vpop                ; st0=accum, st1=term
    cmp r12d, 1
    je .do_div
    ja .do_mod
    fmulp st1, st0
    jmp .loop
.do_div:
    fdivrp st1, st0          ; accum / term
    jmp .loop
.do_mod:
    fprem                    ; знак как у делимого (как fmod)
    fnstsw ax
    test ah, 4
    jnz .do_mod
    fstp st1
    jmp .loop
.fail:
    stc
    pop r12
    ret

unary:
    push r12
    xor r12d, r12d
.loop:
    call skip_ws
    movzx eax, byte [r13]
    cmp al, '+'
    je .plus
    cmp al, '-'
    je .minus
    call power
    jc .fail
    test r12d, r12d
    jz .ok
    fchs
.ok:
    clc
    pop r12
    ret
.plus:
    inc r13
    jmp .loop
.minus:
    inc r13
    xor r12d, 1
    jmp .loop
.fail:
    stc
    pop r12
    ret

power:
    call primary
    jc .fail
.fact:
    call skip_ws
    movzx eax, byte [r13]
    cmp al, '!'
    jne .pow
    inc r13
    call op_fact
    jmp .fact
.pow:
    cmp al, '^'
    jne .done
    inc r13
    call vpush
    jc .fail
    call unary
    jc .fail
    call vpop                ; st0=base, st1=exp
    call do_pow
.done:
    clc
    ret
.fail:
    stc
    ret

; func_arg := ('+'|'-')* primary  — поэтому sin(x)^2 = (sin x)^2
func_arg:
    push r12
    xor r12d, r12d
.loop:
    call skip_ws
    movzx eax, byte [r13]
    cmp al, '+'
    je .plus
    cmp al, '-'
    je .minus
    call primary
    jc .fail
    test r12d, r12d
    jz .ok
    fchs
.ok:
    clc
    pop r12
    ret
.plus:
    inc r13
    jmp .loop
.minus:
    inc r13
    xor r12d, 1
    jmp .loop
.fail:
    stc
    pop r12
    ret

; func_args2 := '(' expr ',' expr ')'  ->  st0=a, st1=b
func_args2:
    call skip_ws
    cmp byte [r13], '('
    jne .fail
    inc r13
    call expr
    jc .fail
    call skip_ws
    cmp byte [r13], ','
    jne .fail
    inc r13
    call vpush
    jc .fail
    call expr
    jc .fail
    call skip_ws
    cmp byte [r13], ')'
    jne .fail
    inc r13
    call vpop
    clc
    ret
.fail:
    stc
    ret

primary:
    call skip_ws
    movzx eax, byte [r13]
    cmp al, '('
    je .paren
    cmp al, '.'
    je .number
    cmp al, '0'
    jb .fail
    cmp al, '9'
    jbe .number
    cmp al, 'a'
    jb .fail
    cmp al, 'z'
    jbe .word
.fail:
    stc
    ret

.number:
    jmp parse_number

.paren:
    inc r13
    call expr
    jc .fail
    call skip_ws
    cmp byte [r13], ')'
    jne .fail
    inc r13
    clc
    ret

.word:
    lea rcx, [functab]
.fl:
    mov rdi, [rcx]
    test rdi, rdi
    jz .consts
    call match_str           ; rcx не портит
    jnc .func
    add rcx, 24
    jmp .fl
.func:
    push qword [rcx + 8]
    cmp qword [rcx + 16], 2
    je .two
    call func_arg
    jmp .call
.two:
    call func_args2
.call:
    pop rax                  ; pop флаги не трогает
    jc .fail
    call rax
    clc
    ret

.consts:
    lea rdi, [k_pi]
    call match_str
    jnc .pi
    lea rdi, [k_e]
    call match_str
    jnc .e
    lea rdi, [k_ans]
    call match_str
    jnc .ans
    ; переменная: одна буква + граница слова
    movzx edx, byte [r13 + 1]
    call is_alnum
    jnc .fail
    movzx ecx, byte [r13]
    sub ecx, 'a'
    bt dword [var_def], ecx
    jnc .fail                ; не задана
    fld qword [vars + rcx*8]
    inc r13
    clc
    ret
.pi:
    fldpi
    clc
    ret
.e:
    fldl2e
    call pow2
    clc
    ret
.ans:
    fld qword [ans]
    clc
    ret

; match_str: rdi -> слово. Граница — следующий символ не [a-z0-9].
; При совпадении сдвигает r13, CF=0. Портит rax, rdx.
match_str:
    push rsi
    push rdi
    mov rsi, r13
.loop:
    movzx eax, byte [rdi]
    test al, al
    jz .matched
    cmp al, [rsi]
    jne .nomatch
    inc rdi
    inc rsi
    jmp .loop
.matched:
    movzx edx, byte [rsi]
    call is_alnum
    jnc .nomatch
    mov r13, rsi
    pop rdi
    pop rsi
    clc
    ret
.nomatch:
    pop rdi
    pop rsi
    stc
    ret

; parse_number: [цифры][.цифры][e[+-]цифры], накопление в x87
parse_number:
    push rbx
    push rcx
    push rdx
    push rsi
    xor ecx, ecx             ; цифр мантиссы
    xor ebx, ebx             ; десятичный порядок
    fldz
.int:
    movzx eax, byte [r13]
    cmp al, '0'
    jb .int_done
    cmp al, '9'
    ja .int_done
    call .add_digit
    inc r13
    inc ecx
    jmp .int
.int_done:
    cmp al, '.'
    jne .mant_done
    inc r13
.frac:
    movzx eax, byte [r13]
    cmp al, '0'
    jb .mant_done
    cmp al, '9'
    ja .mant_done
    call .add_digit
    inc r13
    inc ecx
    dec ebx
    jmp .frac
.mant_done:
    test ecx, ecx
    jz .fail

    cmp byte [r13], 'e'      ; экспонента, только если дальше цифра
    jne .scale
    lea rsi, [r13 + 1]
    xor ecx, ecx
    movzx eax, byte [rsi]
    cmp al, '+'
    je .esign
    cmp al, '-'
    jne .edig0
    mov ecx, 1
.esign:
    inc rsi
.edig0:
    movzx eax, byte [rsi]
    cmp al, '0'
    jb .scale
    cmp al, '9'
    ja .scale
    xor edx, edx
.edig:
    movzx eax, byte [rsi]
    cmp al, '0'
    jb .edone
    cmp al, '9'
    ja .edone
    cmp edx, 100000
    jae .eskip
    imul edx, edx, 10
    sub eax, '0'
    add edx, eax
.eskip:
    inc rsi
    jmp .edig
.edone:
    mov r13, rsi
    test ecx, ecx
    jz .eadd
    neg edx
.eadd:
    add ebx, edx

.scale:
    fldz                     ; 0 * 10^n = 0
    fucomip st0, st1
    je .ok
    test ebx, ebx
    jz .ok
    mov ecx, ebx
    jns .p10_init
    neg ecx
.p10_init:
    cmp ecx, 6000
    jbe .p10_go
    mov ecx, 6000
.p10_go:
    fld1
.p10:
    fmul qword [ten]
    dec ecx
    jnz .p10
    test ebx, ebx
    js .neg_exp
    fmulp st1, st0
    jmp .ok
.neg_exp:
    fdivp st1, st0
.ok:
    clc
    jmp .out
.fail:
    fstp st0
    stc
.out:
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret
.add_digit:                  ; st0 = st0*10 + (al-'0')
    fmul qword [ten]
    sub eax, '0'
    push rax
    fild qword [rsp]
    add rsp, 8
    faddp st1, st0
    ret

; ============================================================
;  Функции. Вход: st0 = x  (или st0 = a, st1 = b), выход: st0
; ============================================================

; --- тригонометрия ---------------------------------------------
op_sin:
    cmp byte [angle_mode], 0
    jne .deg
    fld st0
    call trig_reduce
    fsin
    jmp snap
.deg:
    call deg_reduce          ; r в [-180, 180]
    fld st0
    fabs
    fldz
    fucomip st0, st1
    jp .keep
    je zero2
    fld qword [f_180]
    fucomip st0, st1
    je zero2
    fstp st0
    call deg_to_rad
    fsin
    ret
.keep:
    fstp st0
    ret

op_cos:
    cmp byte [angle_mode], 0
    jne .deg
    fld st0
    call trig_reduce
    fcos
    jmp snap
.deg:
    call deg_reduce
    fld st0
    fabs
    fld qword [f_90]
    fucomip st0, st1
    jp .keep
    je zero2
    fstp st0
    call deg_to_rad
    fcos
    ret
.keep:
    fstp st0
    ret

op_tan:
    cmp byte [angle_mode], 0
    jne .deg
    fld st0
    call trig_reduce
    fptan
    fstp st0
    jmp snap
.deg:
    call deg_reduce
    fld st0
    fabs
    fldz
    fucomip st0, st1
    jp .keep
    je zero2
    fld qword [f_180]
    fucomip st0, st1
    je zero2
    fld qword [f_90]
    fucomip st0, st1
    je nan2                  ; tan(90) не определён
    fstp st0
    call deg_to_rad
    fptan
    fstp st0
    ret
.keep:
    fstp st0
    ret

zero2:                       ; снять 2 значения, вернуть 0
    fstp st0
    fstp st0
    fldz
    ret

nan2:                        ; снять 2 значения, вернуть nan
    fstp st0
    fstp st0
    fld qword [f_nan]
    ret

op_asin:
    fld st0
    fmul st0, st0
    fld1
    fsubrp st1, st0          ; 1 - x^2
    fsqrt
    fpatan
    jmp ang_out

op_acos:
    fld st0
    fmul st0, st0
    fld1
    fsubrp st1, st0
    fsqrt
    fxch st1
    fpatan
    jmp ang_out

op_atan:
    fld1
    fpatan
    jmp ang_out

op_atan2:                    ; st0=y, st1=x
    fxch st1
    fpatan                   ; atan(y/x) с учётом четверти
    jmp ang_out

; радианы -> градусы, если включён deg
ang_out:
    cmp byte [angle_mode], 0
    je .r
    fmul qword [f_180]
    fldpi
    fdivp st1, st0
.r:
    ret

deg_to_rad:
    fldpi
    fmulp st1, st0
    fdiv qword [f_180]
    ret

; st0 = x mod 360 в [-180, 180] — точно
deg_reduce:
    fld qword [f_360]
    fxch st1
.lp:
    fprem1
    fnstsw ax
    test ah, 4
    jnz .lp
    fstp st1
    ret

; fsin/fcos/fptan работают только при |x| < 2^63. Точно свести
; такой x средствами x87 нельзя — отдаём nan.
trig_reduce:
    fld st0
    fabs
    fld qword [f_2p63]
    fucomip st0, st1
    fstp st0                 ; fstp не трогает EFLAGS
    jp .ret
    ja .ret
    fstp st0
    fld qword [f_nan]
.ret:
    ret

; snap: st0 = результат, st1 = x. Если |x| >= 1 и |рез| < |x|*1e-15,
; это шум округления (sin(pi) и т.п.) — возвращаем 0.
snap:
    fld st1
    fabs                     ; |x|, r, x
    fld1
    fucomip st0, st1
    ja .no                   ; |x| < 1
    fmul qword [f_1e_15]     ; t, r, x
    fld st1
    fabs                     ; |r|, t, r, x
    fucomip st0, st1
    fstp st0                 ; r, x
    jp .keep
    jb zero2
.keep:
    fstp st1
    ret
.no:
    fstp st0
    fstp st1
    ret

; --- гиперболические --------------------------------------------
op_sinh:
    fld st0
    fabs
    fld1
    fucomip st0, st1
    fstp st0
    jb .big                  ; |x| > 1
    call expm1               ; u = e^x - 1  (точно при малых x)
    fld st0
    fld1
    faddp st1, st0           ; u+1, u
    fdivr st0, st1           ; u/(u+1), u
    faddp st1, st0
    fmul qword [f_half]
    ret
.big:
    call exp_x               ; E
    fld1
    fdiv st0, st1            ; 1/E, E
    fsubp st1, st0           ; E - 1/E
    fmul qword [f_half]
    ret

op_cosh:
    call exp_x
    fld1
    fdiv st0, st1
    faddp st1, st0
    fmul qword [f_half]
    ret

op_tanh:
    fld st0
    fabs
    fld qword [f_23]
    fucomip st0, st1
    fstp st0
    jp .r
    jb .sat
    fadd st0, st0
    call expm1               ; u = e^2x - 1
    fld st0
    fadd qword [f_two]       ; u+2, u
    fdivp st1, st0           ; u/(u+2)
.r:
    ret
.sat:
    fld1
    fxch st1                 ; x, 1
    jmp apply_sign

op_asinh:                    ; sign(x) * log1p(a + a^2/(1+sqrt(1+a^2)))
    fld st0
    fabs                     ; a, x
    fld st0
    fmul st0, st0            ; a2, a, x
    fld st0
    fld1
    faddp st1, st0
    fsqrt
    fld1
    faddp st1, st0           ; 1+s, a2, a, x
    fdivp st1, st0           ; a2/(1+s), a, x
    faddp st1, st0           ; y, x
    call log1p               ; L, x
    fxch st1
    jmp apply_sign

op_acosh:                    ; ln(x + sqrt(x^2-1))
    fld st0
    fmul st0, st0
    fld1
    fsubp st1, st0
    fsqrt
    faddp st1, st0
    jmp op_ln

op_atanh:                    ; 0.5 * log1p(2x/(1-x))
    fld st0
    fadd st0, st0            ; 2x, x
    fld1
    fsub st0, st2            ; 1-x, 2x, x
    fdivp st1, st0
    fstp st1
    call log1p
    fmul qword [f_half]
    ret

; --- степени, логарифмы ----------------------------------------
op_sqrt:
    fsqrt
    ret

op_cbrt:
    fld st0
    fabs
    fld1
    fxch st1
    fyl2x                    ; log2|x|, x
    fdiv qword [f_three]
    call pow2                ; |x|^(1/3), x
    fxch st1
    jmp apply_sign

op_exp:
exp_x:
    fldl2e
    fmulp st1, st0
    jmp pow2

op_ln:
    fldln2
    fxch st1
    fyl2x
    ret

op_log:
    fldlg2
    fxch st1
    fyl2x
    ret

op_log2:
    fld1
    fxch st1
    fyl2x
    ret

; st0 = ln(1 + y), точно при малых y
log1p:
    fld st0
    fabs
    fld qword [f_l1p]
    fucomip st0, st1
    fstp st0
    jp .gen
    jb .gen
    fldln2
    fxch st1
    fyl2xp1
    ret
.gen:
    fld1
    faddp st1, st0
    jmp op_ln

; st0 = e^x - 1, точно при малых x
expm1:
    fldl2e
    fmulp st1, st0           ; t
    fld st0
    fabs
    fld1
    fucomip st0, st1
    fstp st0
    jp .big
    jb .big
    f2xm1
    ret
.big:
    call pow2
    fld1
    fsubp st1, st0
    ret

; --- округление и прочее ----------------------------------------
op_abs:
    fabs
    ret

op_floor:
    mov ecx, RC_DOWN
    jmp frnd_rc

op_ceil:
    mov ecx, RC_UP
    jmp frnd_rc

op_round:                    ; половина — от нуля
    fld st0
    fabs
    fadd qword [f_half]
    mov ecx, RC_DOWN
    call frnd_rc
    fxch st1
    jmp apply_sign

; frndint с временным режимом округления ecx
frnd_rc:
    sub rsp, 8
    fnstcw [rsp]
    movzx eax, word [rsp]
    mov [rsp + 2], ax
    and eax, 0xF3FF
    or eax, ecx
    mov [rsp + 4], ax
    fldcw [rsp + 4]
    frndint
    fldcw [rsp + 2]
    add rsp, 8
    ret

; st0 = x, st1 = r  ->  r со знаком x
apply_sign:
    fldz
    fucomip st0, st1
    fstp st0
    jbe .r
    fchs
.r:
    ret

op_min:                      ; st0=a, st1=b
    fucomi st0, st1
    ja .b
    fstp st1
    ret
.b:
    fstp st0
    ret

op_max:
    fucomi st0, st1
    jb .b
    fstp st1
    ret
.b:
    fstp st0
    ret

; root(x, n): x^(1/n); при x<0 нужен нечётный целый n
op_root:
    fstp qword [rt_x]
    fstp qword [rt_n]
    fld1
    fdiv qword [rt_n]        ; 1/n
    fld qword [rt_x]         ; x, 1/n
    fldz
    fucomip st0, st1
    jbe do_pow               ; x >= 0 (или nan)
    fld qword [rt_n]
    call is_nni_abs
    jc .nan3
    fisttp qword [tmpq]
    test byte [tmpq], 1
    jz nan2
    fabs
    call do_pow
    fchs
    ret
.nan3:
    fstp st0
    jmp nan2

; is_nni_abs: st0 — целое? (знак не важен). CF=0 — да. Стек не меняет.
is_nni_abs:
    fld st0
    frndint
    fucomip st0, st1
    jp .no
    jne .no
    clc
    ret
.no:
    stc
    ret

; is_nni: st0 — целое >= 0 ? CF=0 — да. Стек не меняет.
is_nni:
    call is_nni_abs
    jc .r
    fldz
    fucomip st0, st1
    ja .no
    clc
.r:
    ret
.no:
    stc
    ret

; n!  (целое n >= 0)
op_fact:
    call is_nni
    jc .nan
    fld qword [f_1800]
    fucomip st0, st1
    jb .inf
    fisttp qword [tmpq]
    mov rcx, [tmpq]
    fld1
    mov edx, 2
.lp:
    cmp rdx, rcx
    ja .done
    mov [tmpq], rdx
    fild qword [tmpq]
    fmulp st1, st0
    inc rdx
    jmp .lp
.done:
    ret
.nan:
    fstp st0
    fld qword [f_nan]
    ret
.inf:
    fstp st0
    fld qword [f_inf]
    ret

; проверка аргументов ncr/npr: st0=n, st1=k -> st0=k, st1=n
; CF=1 — недопустимо (стек не трогаем)
nk_check:
    call is_nni
    jc .bad
    fxch st1
    call is_nni
    jc .bad
    fucomi st0, st1
    ja .bad                  ; k > n
    clc
    ret
.bad:
    stc
    ret

; npr(n,k) = n*(n-1)*...*(n-k+1)
op_npr:
    call nk_check
    jc nan2
    fld qword [f_5000]
    fucomip st0, st1
    jb .inf                  ; k > 5000 -> > 5000! = inf
    fisttp qword [tmpq]      ; n
    mov rcx, [tmpq]
    fld1                     ; acc, n
    test rcx, rcx
    jz .done
.lp:
    fmul st0, st1
    fxch st1
    fld1
    fsubp st1, st0           ; n-1
    fxch st1
    dec rcx
    jnz .lp
.done:
    fstp st1
    ret
.inf:
    fstp st0
    fstp st0
    fld qword [f_inf]
    ret

; ncr(n,k), k = min(k, n-k);  acc = acc*(n-k+i)/i
op_ncr:
    call nk_check
    jc nan2
    fld st1
    fsub st0, st1            ; n-k, k, n
    fucomi st0, st1
    jb .swap
    fstp st0                 ; k, n
    jmp .go
.swap:
    fstp st1                 ; n-k, n
.go:
    fld qword [f_5000]
    fucomip st0, st1
    jb .inf
    fld st0
    fisttp qword [tmpq]
    mov rcx, [tmpq]
    fsubp st1, st0           ; base = n-k
    fld1                     ; acc, base
    mov edx, 1
.lp:
    cmp rdx, rcx
    ja .done
    mov [tmpq], rdx
    fild qword [tmpq]        ; i, acc, base
    fld st0
    fadd st0, st3            ; base+i, i, acc, base
    fmulp st2, st0           ; i, acc*(base+i), base
    fdivp st1, st0           ; acc', base
    inc rdx
    jmp .lp
.done:
    fstp st1
    ret
.inf:
    fstp st0
    fstp st0
    fld qword [f_inf]
    ret

; ------------------------------------------------------------
;  do_pow: st0=base, st1=exp -> base^exp
;  x^0 = 1;  0^(+) = 0;  0^(-) = inf;  (-x)^целое
; ------------------------------------------------------------
do_pow:
    fstp qword [pow_base]
    fstp qword [pow_exp]

    fld qword [pow_exp]
    fldz
    fucomip st0, st1
    fstp st0
    jp .chk_base
    jne .chk_base
    fld1
    ret

.chk_base:
    fld qword [pow_base]
    fldz
    fucomip st0, st1
    fstp st0
    jp .pos
    ja .neg_base
    je .zero_base

.pos:
    fld qword [pow_exp]
    fld qword [pow_base]
    fyl2x
    jmp pow2

.zero_base:
    fld qword [pow_exp]
    fldz
    fucomip st0, st1
    fstp st0
    jp .nan
    ja .inf
    fldz
    ret
.inf:
    fld qword [f_inf]
    ret

.neg_base:
    fld qword [pow_exp]
    call is_nni_abs
    fstp st0
    jc .nan                  ; дробная степень отрицательного

    fld qword [pow_exp]
    fisttp qword [pow_exp_i]
    fld qword [pow_exp]
    fld qword [pow_base]
    fabs
    fyl2x
    call pow2
    test byte [pow_exp_i], 1
    jz .done
    fchs
.done:
    ret

.nan:
    fld qword [f_nan]
    ret

; pow2: st0 = 2^st0 (с обработкой inf и огромных |t|)
pow2:
    fld st0
    fabs
    fld qword [f_big]
    fucomip st0, st1
    fstp st0
    jp .normal
    jae .normal
    fldz
    fucomip st0, st1
    fstp st0
    ja .zero
    fld qword [f_inf]
    ret
.zero:
    fldz
    ret
.normal:
    fld st0
    frndint
    fsub st1, st0
    fxch st1
    f2xm1
    fld1
    faddp st1, st0
    fscale
    fstp st1
    ret

; ============================================================
;  print_num: st0 -> 12 значащих цифр + '\n' (st0 снимается)
;  print_num_at: то же, вывод с адреса rdi внутри outbuf
; ============================================================
print_num:
    lea rdi, [outbuf]
print_num_at:
    push r12
    push r13

    fucomi st0, st0
    jp .nan

    fld st0
    fabs
    fld qword [f_inf]
    fucomip st0, st1
    fstp st0
    je .inf

    fldz
    fucomip st0, st1
    ja .neg
    je .zero
    jmp .pos

.nan:
    fstp st0
    mov dword [rdi], 'nan'
    add rdi, 3
    jmp .finish
.inf:
    fldz
    fucomip st0, st1
    fstp st0
    jbe .pinf
    mov byte [rdi], '-'
    inc rdi
.pinf:
    mov dword [rdi], 'inf'
    add rdi, 3
    jmp .finish
.zero:
    fstp st0
    mov byte [rdi], '0'
    inc rdi
    jmp .finish
.neg:
    mov byte [rdi], '-'
    inc rdi
    fchs

.pos:                        ; нормализация в [1, 10), r13 = порядок
    xor r13d, r13d
.hi:
    fld qword [ten]
    fucomip st0, st1
    ja .lo
    fdiv qword [ten]
    inc r13d
    jmp .hi
.lo:
    fld1
    fucomip st0, st1
    jbe .norm
    fmul qword [ten]
    dec r13d
    jmp .lo
.norm:
    fmul qword [f_1e11]
    frndint
    fistp qword [tmpq]
    mov rax, [tmpq]
    mov rcx, 1000000000000
    cmp rax, rcx
    jb .digits
    mov rax, 100000000000    ; 9.99..95 -> 10
    inc r13d
.digits:                     ; 12 цифр в dbuf
    lea rsi, [dbuf + 12]
    mov r8, 10
    mov ecx, 12
.dg:
    xor edx, edx
    div r8
    add dl, '0'
    dec rsi
    mov [rsi], dl
    dec ecx
    jnz .dg
    mov ecx, 12              ; ecx = значащих цифр без хвостовых нулей
.tz:
    cmp ecx, 1
    jbe .fmt
    cmp byte [dbuf + rcx - 1], '0'
    jne .fmt
    dec ecx
    jmp .tz

.fmt:
    cmp r13d, 12
    jge .sci
    cmp r13d, -5
    jl .sci
    test r13d, r13d
    js .small
    lea edx, [r13 + 1]       ; цифр в целой части
    xor esi, esi
.ip:
    cmp esi, edx
    jae .ipd
    mov al, '0'
    cmp esi, ecx
    jae .ipz
    mov al, [dbuf + rsi]
.ipz:
    mov [rdi], al
    inc rdi
    inc esi
    jmp .ip
.ipd:
    cmp ecx, edx
    jbe .finish
    mov byte [rdi], '.'
    inc rdi
.fp:
    mov al, [dbuf + rsi]
    mov [rdi], al
    inc rdi
    inc esi
    cmp esi, ecx
    jb .fp
    jmp .finish

.small:                      ; 0.000ddd
    mov word [rdi], '0.'
    add rdi, 2
    mov edx, r13d
    neg edx
    dec edx
.sz:
    test edx, edx
    jz .szd
    mov byte [rdi], '0'
    inc rdi
    dec edx
    jmp .sz
.szd:
    xor esi, esi
.sd:
    mov al, [dbuf + rsi]
    mov [rdi], al
    inc rdi
    inc esi
    cmp esi, ecx
    jb .sd
    jmp .finish

.sci:                        ; d.ddde+NN
    mov al, [dbuf]
    mov [rdi], al
    inc rdi
    cmp ecx, 1
    jbe .se
    mov byte [rdi], '.'
    inc rdi
    mov esi, 1
.sf:
    mov al, [dbuf + rsi]
    mov [rdi], al
    inc rdi
    inc esi
    cmp esi, ecx
    jb .sf
.se:
    mov byte [rdi], 'e'
    inc rdi
    mov eax, r13d
    mov byte [rdi], '+'
    test eax, eax
    jns .sep
    mov byte [rdi], '-'
    neg eax
.sep:
    inc rdi
    call put_u64

.finish:
    mov byte [rdi], 10
    inc rdi
    lea rsi, [outbuf]
    mov rdx, rdi
    sub rdx, rsi
    mov eax, 1
    mov edi, 1
    syscall
    pop r13
    pop r12
    ret

; put_u64: rax -> десятичная строка в rdi
put_u64:
    push rbx
    push r8
    push r9
    lea rbx, [intbuf + 40]
    mov byte [rbx], 0
    mov r9, rax
    mov r8, 10
    test r9, r9
    jnz .l
    dec rbx
    mov byte [rbx], '0'
    jmp .c
.l:
    mov rax, r9
    xor edx, edx
    div r8
    mov r9, rax
    add dl, '0'
    dec rbx
    mov [rbx], dl
    test r9, r9
    jnz .l
.c:
    mov al, [rbx]
    test al, al
    jz .d
    mov [rdi], al
    inc rdi
    inc rbx
    jmp .c
.d:
    pop r9
    pop r8
    pop rbx
    ret
