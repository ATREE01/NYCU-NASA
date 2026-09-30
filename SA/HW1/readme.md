Wordle/Quordle

```bash
#!/bin/sh

# 任務伺服器端點與個人設定
API_ENDPOINT="http://192.168.255.69"
STUID=$((254 * 256 + 145)) # 請替換成你的學號
DICT_FILE="dictionary.txt"

# 輸出 Help 訊息到 stderr
print_usage() {
cat >&2 << 'EOF'
hw1.sh -t TASK_TYPE [-h]

Available Options:
-t SYS_INFO | WORDLE | QUORDLE : Task type
-h : Show the script usage
EOF
}

cleanup_tasks() {
    echo "[*] Cleaning up existing tasks..." >&2
    TASKS_JSON=$(curl -s "$API_ENDPOINT/tasks/stu/$STUID")

    if echo "$TASKS_JSON" | grep -q '"id"'; then
        for tid in $(echo "$TASKS_JSON" | jq -r '.tasks[].id // empty'); do
            curl -s -X DELETE "$API_ENDPOINT/tasks/$tid" > /dev/null
        done
    fi
}

ensure_dictionary() {
    if [ ! -s "$DICT_FILE" ]; then
        echo "[*] Downloading dictionary..." >&2
        curl -s "$API_ENDPOINT/dictionary" > "$DICT_FILE"
    fi
}

# 核心過濾邏輯
filter_words() {
    guess=$1
    result=$2
    infile=$3
    outfile=$4

    cp "$infile" "$outfile.tmp"

    for pos in 1 2 3 4 5; do
        char=$(echo "$guess" | cut -c "$pos")
        res=$(echo "$result" | cut -c "$pos")

        if [ "$res" = "A" ]; then
            regex="^"
            for j in 1 2 3 4 5; do
                if [ "$j" -eq "$pos" ]; then regex="${regex}${char}"; else regex="${regex}."; fi
            done
            grep -E "$regex" "$outfile.tmp" > "$outfile.tmp2"
            mv "$outfile.tmp2" "$outfile.tmp"

        elif [ "$res" = "B" ]; then
            grep "$char" "$outfile.tmp" > "$outfile.tmp2"
            mv "$outfile.tmp2" "$outfile.tmp"

            regex="^"
            for j in 1 2 3 4 5; do
                if [ "$j" -eq "$pos" ]; then regex="${regex}[^${char}]"; else regex="${regex}."; fi
            done
            grep -E "$regex" "$outfile.tmp" > "$outfile.tmp2"
            mv "$outfile.tmp2" "$outfile.tmp"

        elif [ "$res" = "X" ]; then
            regex="^"
            for j in 1 2 3 4 5; do
                if [ "$j" -eq "$pos" ]; then regex="${regex}[^${char}]"; else regex="${regex}."; fi
            done
            grep -E "$regex" "$outfile.tmp" > "$outfile.tmp2"
            mv "$outfile.tmp2" "$outfile.tmp"

            has_ab=0
            for k in 1 2 3 4 5; do
                k_char=$(echo "$guess" | cut -c "$k")
                k_res=$(echo "$result" | cut -c "$k")
                if [ "$k_char" = "$char" ] && { [ "$k_res" = "A" ] || [ "$k_res" = "B" ]; }; then
                    has_ab=1
                fi
            done
            if [ "$has_ab" -eq 0 ]; then
                grep -v "$char" "$outfile.tmp" > "$outfile.tmp2"
                mv "$outfile.tmp2" "$outfile.tmp"
            fi
        fi
    done
    mv "$outfile.tmp" "$outfile"
}

fetch_valid_task() {
    req_type=$1
    while true; do
        echo "[*] Requesting $req_type task..."
        RESPONSE=$(curl -s -X POST "$API_ENDPOINT/tasks" -H "Content-Type: application/json" -d "{\"stuid\": \"$STUID\", \"type\": \"$req_type\"}")
        tid=$(echo "$RESPONSE" | jq -r '.id // empty')

        if [ -z "$tid" ]; then
            echo "[-] Failed to get Task ID. Retrying..."
            cleanup_tasks
            continue
        fi

        tstatus=$(echo "$RESPONSE" | jq -r '.status // empty')
        if [ "$tstatus" = "TIMEOUT" ]; then
            echo "[-] Got a TIMEOUT task ($tid). Deleting and retrying..."
            curl -s -X DELETE "$API_ENDPOINT/tasks/$tid" > /dev/null
            cleanup_tasks
            continue
        elif [ "$tstatus" = "PENDING" ]; then
            echo "[+] Task ID: $tid"
            TASK_ID="$tid"
            break
        else
            echo "[-] Unexpected status: $tstatus. Retrying..."
            curl -s -X DELETE "$API_ENDPOINT/tasks/$tid" > /dev/null
            continue
        fi
    done
}

TASK_TYPE=""

while getopts ":t:h" opt; do
    case "$opt" in
        t) TASK_TYPE="$OPTARG" ;;
        h) print_usage; exit 0 ;;
        ?) print_usage; exit 1 ;;
    esac
done

if [ -z "$TASK_TYPE" ]; then
    print_usage
    exit 1
fi

case "$TASK_TYPE" in
    SYS_INFO)
        os_name=$(grep -E '^PRETTY_NAME=' /etc/os-release | cut -d '"' -f 2)
        arch=$(uname -m)
        echo "OS: ${os_name} ${arch}"
        echo "Kernel: $(uname -r)"
        shell_name=$(ps -p $$ -o comm=)
        echo "Shell: ${shell_name}"
        echo "Terminal: $(tty)"
        cpu_model=$(awk -F': ' '/model name/ {print $2; exit}' /proc/cpuinfo)
        echo "CPU: ${cpu_model}"
        ;;

    WORDLE)
        cleanup_tasks
        ensure_dictionary
        fetch_valid_task "WORDLE"

        # 修正重點：複製檔案時，一律轉成大寫！
        tr '[:lower:]' '[:upper:]' < "$DICT_FILE" > "candidates_${TASK_ID}.txt"

        guess_count=1
        while [ "$guess_count" -le 10 ]; do
            # 檔案已經是大寫了，尋找 CRANE 也要用大寫去抓
            if [ "$guess_count" -eq 1 ] && grep -q "^CRANE$" "candidates_${TASK_ID}.txt"; then
                GUESS="CRANE"
            else
                # 這裡也不需要再 tr 轉大寫了
                GUESS=$(head -n 1 "candidates_${TASK_ID}.txt")
            fi

            if [ -z "$GUESS" ]; then
                echo "[-] Run out of words!"
                exit 1
            fi

            echo "[*] Guess $guess_count: $GUESS"

            SUBMIT_RES=$(curl -s -X POST "$API_ENDPOINT/tasks/$TASK_ID/submit" -H "Content-Type: application/json" -d "{\"answer\": \"$GUESS\"}")
            PROBLEM=$(echo "$SUBMIT_RES" | jq -r '.problem // empty')

            if [ -z "$PROBLEM" ]; then
                TASK_INFO=$(curl -s "$API_ENDPOINT/tasks/$TASK_ID")
                STATUS=$(echo "$TASK_INFO" | jq -r '.status')
                if [ "$STATUS" = "SOLVED" ]; then echo "[+] Solved!"; break; fi
                if [ "$STATUS" = "FAILED" ]; then echo "[-] Failed!"; break; fi
            fi

            echo "    -> Result: $PROBLEM"
            if [ "$PROBLEM" = "AAAAA" ]; then
                echo "[+] WORDLE Solved! Answer: $GUESS"
                break
            fi

            filter_words "$GUESS" "$PROBLEM" "candidates_${TASK_ID}.txt" "candidates_${TASK_ID}.txt"
            guess_count=$((guess_count + 1))
        done
        rm -f "candidates_${TASK_ID}.txt"
        ;;

    QUORDLE)
        cleanup_tasks
        ensure_dictionary
        fetch_valid_task "QUORDLE"

        # 建立 Quordle 候選名單，轉大寫
        for i in 1 2 3 4; do tr '[:lower:]' '[:upper:]' < "$DICT_FILE" > "q_cand_${i}.txt"; done

        solved1=0; solved2=0; solved3=0; solved4=0
        guess_count=1

        while [ "$guess_count" -le 20 ]; do
            GUESS=""
            if [ $solved1 -eq 0 ]; then GUESS=$(head -n 1 "q_cand_1.txt");
            elif [ $solved2 -eq 0 ]; then GUESS=$(head -n 1 "q_cand_2.txt");
            elif [ $solved3 -eq 0 ]; then GUESS=$(head -n 1 "q_cand_3.txt");
            elif [ $solved4 -eq 0 ]; then GUESS=$(head -n 1 "q_cand_4.txt");
            fi

            # 使用第一個檔案檢查字典裡是否有 CRANE（大寫）
            if [ "$guess_count" -eq 1 ] && grep -q "^CRANE$" "q_cand_1.txt"; then
                GUESS="CRANE"
            fi

            if [ -z "$GUESS" ]; then
                echo "[-] Run out of words!"
                break
            fi

            echo "[*] Guess $guess_count: $GUESS"
            SUBMIT_RES=$(curl -s -X POST "$API_ENDPOINT/tasks/$TASK_ID/submit" -H "Content-Type: application/json" -d "{\"answer\": \"$GUESS\"}")

            PROB1=$(echo "$SUBMIT_RES" | jq -r '.problem1 // empty')
            if [ -z "$PROB1" ]; then
                TASK_INFO=$(curl -s "$API_ENDPOINT/tasks/$TASK_ID")
                STATUS=$(echo "$TASK_INFO" | jq -r '.status')
                if [ "$STATUS" = "SOLVED" ]; then echo "[+] QUORDLE Solved!"; break; fi
                if [ "$STATUS" = "FAILED" ]; then echo "[-] Task FAILED!"; break; fi
            fi

            PROB2=$(echo "$SUBMIT_RES" | jq -r '.problem2 // empty')
            PROB3=$(echo "$SUBMIT_RES" | jq -r '.problem3 // empty')
            PROB4=$(echo "$SUBMIT_RES" | jq -r '.problem4 // empty')

            # === 新增：將 4 個版面的狀態清楚印出 ===
            disp1="$PROB1"; [ $solved1 -eq 1 ] && disp1="[DONE]"
            disp2="$PROB2"; [ $solved2 -eq 1 ] && disp2="[DONE]"
            disp3="$PROB3"; [ $solved3 -eq 1 ] && disp3="[DONE]"
            disp4="$PROB4"; [ $solved4 -eq 1 ] && disp4="[DONE]"

            echo "    -> Q1: $disp1 | Q2: $disp2 | Q3: $disp3 | Q4: $disp4"
            # ======================================

            if [ "$PROB1" = "AAAAA" ]; then solved1=1; fi
            if [ "$PROB2" = "AAAAA" ]; then solved2=1; fi
            if [ "$PROB3" = "AAAAA" ]; then solved3=1; fi
            if [ "$PROB4" = "AAAAA" ]; then solved4=1; fi

            if [ $solved1 -eq 1 ] && [ $solved2 -eq 1 ] && [ $solved3 -eq 1 ] && [ $solved4 -eq 1 ]; then
                echo "[+] QUORDLE Solved successfully!"
                break
            fi

            if [ $solved1 -eq 0 ]; then filter_words "$GUESS" "$PROB1" "q_cand_1.txt" "q_cand_1.txt"; fi
            if [ $solved2 -eq 0 ]; then filter_words "$GUESS" "$PROB2" "q_cand_2.txt" "q_cand_2.txt"; fi
            if [ $solved3 -eq 0 ]; then filter_words "$GUESS" "$PROB3" "q_cand_3.txt" "q_cand_3.txt"; fi
            if [ $solved4 -eq 0 ]; then filter_words "$GUESS" "$PROB4" "q_cand_4.txt" "q_cand_4.txt"; fi

            guess_count=$((guess_count + 1))
        done

        rm -f q_cand_*.txt
        ;;

    *)
        print_usage
        exit 1
        ;;
esac
```