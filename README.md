# COAL-CCP
The Remote Data System is an Assembly Language–based project developed using EMU8086 that demonstrates low-level programming concepts of the Intel 8086 processor. 
.model small
.stack 100h
.data
    my_file      db "store.csv", 0
    fid          dw ?
    
    ; --- Input Buffers ---
    raw_in       db 20, ?, 20 dup('$') ; General input
    search_buf   db 20, ?, 20 dup('$') ; Buffer for ID to find
    char_buf     db ?

    ; --- Menus and Messages (UPDATED ORDER) ---
    txt_opt      db 13,10,13,10,"=== INVENTORY SYSTEM ==="
                 db 13,10,"1. Insert Data"
                 db 13,10,"2. View Data"
                 db 13,10,"3. Sort by ID (View Only)"
                 db 13,10,"4. Delete by ID"
                 db 13,10,"5. Update Record"
                 db 13,10,"6. Quit"       ; <--- Moved to bottom
                 db 13,10,"Select: $"

    p_id         db 13,10,"ID#: $"
    p_name       db 13,10,"Name: $"
    p_mem        db 13,10,"Members: $"
    p_h2o        db 13,10,"Water: $"
    p_att        db 13,10,"Flour: $"
    p_dal        db 13,10,"Pulses: $"
    
    saved_txt    db 13,10,"Record Saved.$"
    view_hdr     db 13,10,"--- FILE CONTENTS ---", 13, 10, "$"
    sort_msg     db 13,10,"Sorting records by ID...", 13, 10, "$"
    
    ask_del      db 13,10,"Enter ID to Delete: $"
    ask_upd      db 13,10,"Enter ID to Update: $"
    upd_found    db 13,10,"ID Found. Enter new details:", 13, 10, "$"
    del_done     db 13,10,"Operation Complete.$"
    
    sep          db ","
    end_line     db 13,10

    ; --- File Processing Variables ---
    file_buffer  db 4000 dup(0)   
    file_size    dw 0             
    line_ptrs    dw 100 dup(0)    
    rec_count    dw 0             
    
.code
main proc
    mov ax, @data
    mov ds, ax

    ; Check/Create file if not exists
    mov ah, 3dh
    mov al, 0
    lea dx, my_file
    int 21h
    jnc close_init
    
    mov ah, 3ch
    mov cx, 0
    lea dx, my_file
    int 21h
    
close_init:
    mov bx, ax
    mov ah, 3eh
    int 21h

start_app:
    lea dx, txt_opt
    mov ah, 09h
    int 21h
    
    mov ah, 01h
    int 21h
    
    ; --- UPDATED MENU LOGIC ---
    cmp al, '1'
    je  goto_insert
    cmp al, '2'
    je  goto_view
    cmp al, '3'
    je  goto_sort     ; Now 3 is Sort
    cmp al, '4'
    je  goto_delete   ; Now 4 is Delete
    cmp al, '5'
    je  goto_update   ; Now 5 is Update
    cmp al, '6'
    je  kill_app      ; Now 6 is Quit
    jmp start_app

; ============================
; 1. INSERT DATA
; ============================
goto_insert:
    mov ah, 3dh
    mov al, 2
    lea dx, my_file
    int 21h
    mov fid, ax
    
    mov ah, 42h
    mov al, 2
    mov bx, fid
    xor cx, cx
    xor dx, dx
    int 21h
    
    lea dx, p_id
    call get_input_write
    call write_sep
    
    call perform_data_entry
    
    mov ah, 3eh
    mov bx, fid
    int 21h
    
    lea dx, saved_txt
    mov ah, 09h
    int 21h
    jmp start_app

; ============================
; 2. VIEW DATA
; ============================
goto_view:
    lea dx, view_hdr
    mov ah, 09h
    int 21h
    
    mov ah, 3dh
    mov al, 0
    lea dx, my_file
    int 21h
    mov fid, ax
    
read_char_loop:
    mov ah, 3fh
    mov bx, fid
    mov cx, 1
    lea dx, char_buf
    int 21h
    cmp ax, 0
    je close_view
    
    mov dl, char_buf
    mov ah, 02h
    int 21h
    jmp read_char_loop
    
close_view:
    mov ah, 3eh
    mov bx, fid
    int 21h
    jmp start_app

; ============================
; 3. SORT BY ID (View Only)
; ============================
goto_sort:
    lea dx, sort_msg
    mov ah, 09h
    int 21h

    call read_file_to_buffer
    cmp ax, 0
    je start_app

    call parse_lines_to_ptrs
    
    mov cx, rec_count
    dec cx
    cmp cx, 0
    jle print_sorted_data
    
outer_sort:
    push cx
    lea bx, line_ptrs
inner_sort:
    mov si, [bx]
    mov di, [bx+2]
    
    mov al, [si]
    mov dl, [di]
    cmp al, dl
    jbe no_swap
    
    mov ax, [bx]
    xchg ax, [bx+2]
    mov [bx], ax
    
no_swap:
    add bx, 2
    loop inner_sort
    pop cx
    loop outer_sort

print_sorted_data:
    lea dx, view_hdr
    mov ah, 09h
    int 21h

    lea bx, line_ptrs
    mov cx, rec_count
print_loop_sort:
    push cx
    mov si, [bx]
char_out_sort:
    mov dl, [si]
    cmp dl, 10
    je print_lf_sort
    
    mov ah, 02h
    int 21h
    inc si
    jmp char_out_sort
print_lf_sort:
    mov dl, 10
    mov ah, 02h
    int 21h
    add bx, 2
    pop cx
    loop print_loop_sort
    jmp start_app

; ============================
; 4. DELETE BY ID
; ============================
goto_delete:
    lea dx, ask_del
    mov ah, 09h
    int 21h
    
    lea dx, search_buf
    mov ah, 0ah
    int 21h
    
    call read_file_to_buffer
    cmp ax, 0
    je start_app
    
    mov ah, 3ch
    mov cx, 0
    lea dx, my_file
    int 21h
    mov fid, ax

    call process_file_rewrite_delete
    jmp done_rewrite

; ============================
; 5. UPDATE RECORD
; ============================
goto_update:
    lea dx, ask_upd
    mov ah, 09h
    int 21h
    
    lea dx, search_buf
    mov ah, 0ah
    int 21h
    
    call read_file_to_buffer
    cmp ax, 0
    je start_app
    
    mov ah, 3ch
    mov cx, 0
    lea dx, my_file
    int 21h
    mov fid, ax
    
    call process_file_rewrite_update
    
done_rewrite:
    mov ah, 3eh
    mov bx, fid
    int 21h
    lea dx, del_done
    mov ah, 09h
    int 21h
    jmp start_app

; ============================
; 6. QUIT
; ============================
kill_app:
    mov ah, 4ch
    int 21h

main endp

; ============================
; PROCEDURES
; ============================

process_file_rewrite_delete proc
    lea si, file_buffer
    mov di, si
    mov bx, file_size
    add bx, offset file_buffer

scan_lines_del:
    cmp si, bx
    jae ret_proc_del

    push bx
    push di
    lea bx, search_buf+2
    mov ch, 0
    mov cl, search_buf+1
    
check_match_del:
    mov al, [di]
    mov ah, [bx]
    cmp al, ah
    jne no_match_del
    inc di
    inc bx
    dec cl
    jnz check_match_del
    
    cmp byte ptr [di], ',' 
    jne no_match_del
    
    pop di
    pop bx
skip_loop_del:
    cmp byte ptr [si], 10
    je found_skip_del
    inc si
    jmp skip_loop_del
found_skip_del:
    inc si
    mov di, si
    jmp scan_lines_del

no_match_del:
    pop di
    pop bx
write_loop_del:
    mov al, [si]
    call write_char_to_file
    cmp byte ptr [si], 10
    je line_done_del
    inc si
    jmp write_loop_del
line_done_del:
    inc si
    mov di, si
    jmp scan_lines_del
    
ret_proc_del:
    ret
process_file_rewrite_delete endp

process_file_rewrite_update proc
    lea si, file_buffer
    mov di, si
    mov bx, file_size
    add bx, offset file_buffer

scan_lines_upd:
    cmp si, bx
    jae ret_proc_upd

    push bx
    push di
    lea bx, search_buf+2
    mov ch, 0
    mov cl, search_buf+1
    
check_match_upd:
    mov al, [di]
    mov ah, [bx]
    cmp al, ah
    jne no_match_upd
    inc di
    inc bx
    dec cl
    jnz check_match_upd
    
    cmp byte ptr [di], ','
    jne no_match_upd
    
    pop di
    pop bx
    
    ; Write ID
    lea dx, search_buf+2
    mov cl, search_buf+1
    mov ch, 0
    mov ah, 40h
    mov bx, fid
    int 21h
    
    call write_sep
    
    lea dx, upd_found
    mov ah, 09h
    int 21h
    
    call perform_data_entry 
    
skip_loop_upd:
    cmp byte ptr [si], 10
    je found_skip_upd
    inc si
    jmp skip_loop_upd
found_skip_upd:
    inc si
    mov di, si
    jmp scan_lines_upd

no_match_upd:
    pop di
    pop bx
write_loop_upd:
    mov al, [si]
    call write_char_to_file
    cmp byte ptr [si], 10
    je line_done_upd
    inc si
    jmp write_loop_upd
line_done_upd:
    inc si
    mov di, si
    jmp scan_lines_upd
    
ret_proc_upd:
    ret
process_file_rewrite_update endp

perform_data_entry proc
    lea dx, p_name
    call get_input_write
    call write_sep

    lea dx, p_mem
    call get_input_write
    call write_sep

    lea dx, p_h2o
    call get_input_write
    call write_sep

    lea dx, p_att
    call get_input_write
    call write_sep

    lea dx, p_dal
    call get_input_write
    
    lea dx, end_line
    mov cx, 2
    mov ah, 40h
    int 21h
    ret
perform_data_entry endp

write_char_to_file proc

    push ax
    push bx
    push cx
    push dx

    mov char_buf, al

    mov ah, 40h    
    mov bx, fid    
    mov cx, 1        
    lea dx, char_buf  
    int 21h          

    pop dx
    pop cx
    pop bx
    pop ax
    ret
write_char_to_file endp

get_input_write proc
    mov ah, 09h
    int 21h
    lea dx, raw_in
    mov ah, 0ah
    int 21h
    mov bx, fid
    mov ch, 0
    mov cl, raw_in+1
    lea dx, raw_in+2
    mov ah, 40h
    int 21h
    ret
get_input_write endp

write_sep proc
    mov bx, fid
    lea dx, sep
    mov cx, 1
    mov ah, 40h
    int 21h
    ret
write_sep endp

read_file_to_buffer proc
    mov ah, 3dh
    mov al, 0
    lea dx, my_file
    int 21h
    mov fid, ax
    
    mov ah, 3fh
    mov bx, fid
    mov cx, 4000
    lea dx, file_buffer
    int 21h
    mov file_size, ax
    
    push ax
    mov ah, 3eh
    mov bx, fid
    int 21h
    pop ax
    ret
read_file_to_buffer endp

parse_lines_to_ptrs proc
    lea di, file_buffer
    lea bx, line_ptrs
    mov [bx], di
    mov rec_count, 1
    mov cx, file_size
    add cx, offset file_buffer
parse_loop:
    cmp di, cx
    jae parse_done
    cmp byte ptr [di], 10
    jne next_char
    inc di
    cmp di, cx
    jae parse_done
    add bx, 2
    mov [bx], di
    inc rec_count
    jmp parse_loop
next_char:
    inc di
    jmp parse_loop
parse_done:
    ret
parse_lines_to_ptrs endp

end main
