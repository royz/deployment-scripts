### An interative menu within terminal to select which app to deploy and then run corresponding deploy script/commands

> Add these functions to `.bashrc` file, reload bash and then run `deploy`

```bash
# function to echo given string with a color
function print() {
    case "$2" in
    BLACK) n=30 ;;
    RED) n=31 ;;
    GREEN) n=32 ;;
    YELLOW) n=33 ;;
    BLUE) n=34 ;;
    PURPLE) n=35 ;;
    CYAN) n=36 ;;
    WHITE) n=37 ;;
    esac
    echo -e "\033[${n}m$1\033[0m"
}

# takes a list of inputs, lets you select one option, sets the result to a variable
function choose_from_list() {
    local prompt="$1" outvar="$2"
    shift
    shift
    local options=("$@") cur=0 count=${#options[@]} index=0
    local esc=$(echo -en "\e") # cache ESC as test doesn't allow esc codes
    printf "$prompt\n"
    while true; do
        # list all options (option list is zero-based)
        index=0
        for o in "${options[@]}"; do
            if [ "$index" == "$cur" ]; then
                echo -e " >\e[7m$o\e[0m" # mark & highlight the current option
            else
                echo "  $o"
            fi
            ((index++))
        done
        read -s -n3 key               # wait for user to key in arrows or ENTER
        if [[ $key == $esc[A ]]; then # up arrow
            ((cur--))
            ((cur < 0)) && ((cur = 0))
        elif [[ $key == $esc[B ]]; then # down arrow
            ((cur++))
            ((cur >= count)) && ((cur = count - 1))
        elif [[ $key == "" ]]; then # nothing, i.e the read delimiter - ENTER
            break
        fi
        echo -en "\e[${count}A" # go up to the beginning to re-render
    done
    printf -v $outvar "${options[$cur]}"
}

function deploy() {
    selections=(
        "App 1"
        "App 2"
    )
    choose_from_list "Select an app to deploy:" selected_choice "${selections[@]}"
    echo "Runnging deploy script for: $selected_choice"

    PROJECT_DIR=""
    if [ "$selected_choice" = "App 1" ]; then
        PROJECT_DIR="${HOME}/projects/<app-1-path>"
    elif [ "$selected_choice" = "App 2" ]; then
        PROJECT_DIR="${HOME}/projects/<app-2-path>"
    fi

    print "Project dir: $PROJECT_DIR" CYAN

    if [ $PROJECT_DIR = "" ]; then
        print "Invalid selection" RED
        exit 0
    fi

    if [ ! -d $PROJECT_DIR ]; then
        print "$PROJECT_DIR does not exist!" RED
        exit 0
    fi

    cd $PROJECT_DIR
    git add .
    git stash
    git pull
    . deploy.sh
}
```
