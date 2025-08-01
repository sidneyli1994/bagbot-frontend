<script>
	import { onMount, afterUpdate } from 'svelte';  // Importar onMount
    import MessageBlock from './MessageBlock.svelte';
    import { user, userid, isLoggedIn, showOptions, selectedOption } from '../store.js'; // para saber si es logueado o invitado
    export let selectedDate = null;
    export let locked = false; // Cuando locked==true, deshabilita input/botón        
    
    let loggedIn; //Esta logueado?
    let userId; //id del usuario
    let name = ''; //name
    let userMessage = "";    // Input del usuario
    let inputRef; // referencia al input
	let chat = [];      // Arreglo para almacenar la conversación

    let chatEnd = null; //Carga el final del chat

    isLoggedIn.subscribe(value => { loggedIn = value; });
	user.subscribe(value => { name = value; });
    userid.subscribe(value => { userId = value; });

    let userName = name;
    let chatInit = [
    {
        "type": 1,
        "message": "👋 Hola"+(userName ? " " + userName.split(" ")[0] : "")+", mi nombre es Bagbot y soy tu asistente de biblioteca virtual, por favor sé lo más claro y específico posible. Estoy aquí para ayudarte,<br> ¿Qué deseas hacer hoy?"
    }];

    let showOpt;
	showOptions.subscribe(value => { showOpt = value; });
    let userOption;
	selectedOption.subscribe(value => { userOption = value; });

    // Cargar mensajes si seleccionan una fecha
    $: if (selectedDate) {
        getChat_DB_byDate(selectedDate);
    }
    async function getChat_DB_byDate(date) {
        const res = await fetch(`https://bagbot-backend.onrender.com/query/${date}`);
        chat = await res.json();
    }
	// Función para enviar el mensaje al JSON
    async function sendMessage_JSON() {
        const response = await fetch('https://bagbot-backend.onrender.com/send-message-json', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({ message: userMessage })
        });

        const data = await response.json();
        chat = data;  // Actualizar la conversación con la respuesta del servidor
        userMessage = "";  // Limpiar el input después de enviar el mensaje
    }
    //Para guardarlo en la bd
    async function sendMessage_DB() {
        const data = { userId, name, userMessage };
        try {
        const res = await fetch('https://bagbot-backend.onrender.com/send-message-db', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(data)
        });

        if (res.ok) {
            await getChat_DB(false);
            userMessage = "";  // Limpiar el input después de enviar el mensaje
        } else {
            const err = await res.json();
            console.log(err)
        }
        } catch (err) {
            console.log(err)
        }
    }
    // Función para obtener la conversación almacenada en el JSON
    async function getChat_JSON() {
        const respJ = await fetch('https://bagbot-backend.onrender.com/get-chat-json');
        const dataJ = await respJ.json();
        chat = dataJ;
    }

    // Función para obtener la conversación almacenada en bd
    async function getChat_DB(firstTime) {
        const resp = await fetch('https://bagbot-backend.onrender.com/get-chat-db');
        const data = await resp.json();
        chat = data;
            if (chat.length > 0 && firstTime){
                chat.push({'type': 1, 'message': '👋 Bievenid@ de nuevo '+userName+', mi nombre es Bagbot y soy tu asistente de biblioteca virtual, por favor sé lo más claro y específico posible. Estoy aquí para ayudarte,<br>¿Qué deseas hacer hoy? <br>'})
            }
    }

    // Función para detectar la tecla Enter y enviar el mensaje
    async function handleKeyPress(event) {
        if (event.key === 'Enter') {
            sendMessage(); // Llama a la función para enviar el mensaje
        }
    }

// Either afterUpdate()
    afterUpdate(() => {
		if(chat) scrollToBottom(chatEnd);
    });

    $: if(chat && chatEnd) {
		scrollToBottom(chatEnd);
	}
    //Cargar al final del chat
    const scrollToBottom = async (node) => {
        node.scroll({ top: node.scrollHeight, behavior: 'smooth' });
    };
    //Enviar mensaje, aqui si esta logueado se guarda en la bd, sino en el archivo json
    function sendMessage() {
        if (!userMessage.trim()) {
        // Si el mensaje está vacío o solo tiene espacios
        inputRef.focus(); // hacer focus en el input
        return;
        }
        if (loggedIn){
            sendMessage_DB();
        }else{
            sendMessage_JSON();
        }
    }
    onMount(() => {
        if (loggedIn){
            //Si esta logueado obtenemos la conversacion de la BD
            getChat_DB(true); //true porque es primera vez que carga
        }else{
            //Sino obtenemos la conversacion del JSON
            getChat_JSON();
        }
        scrollToBottom(); // Scroll inicial al montar
    });
    let option = '';
    function optionSelected(event) {
		option = event.detail.option;
        sendOptionSelected(option);
	}
    async function sendOptionSelected(option) {
        if (loggedIn){
            let first;
            if (chat.length > 0){
                first = 0;
            }else{
                first=1;
            }
            const values = { userId, name, userMessage,option, first};
            //Si esta logueado obtenemos la conversacion de la BD
            await fetch('https://bagbot-backend.onrender.com/selected-option-chat-db', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(values)
            });
            getChat_DB(false);
        }else{
            await fetch('https://bagbot-backend.onrender.com/selected-option-chat-json', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({ option: option })
            });
            //Sino obtenemos la conversacion del JSON
            getChat_JSON();
        }
    }

    let file;
    let filename = '';
    let errorPDF = '';
    let showButtonDownload = false;

    async function uploadPDF() {
        if (!file) {
            errorPDF = 'Debes seleccionar un archivo PDF.';
            setTimeout(() => {errorPDF = ''; }, 5000);
        return;
        }
        const formData = new FormData();
        formData.append('file', file);
        formData.append('userID', userId); // si necesitas enviar esto
        formData.append('loggedIn', loggedIn);
        try {
        const res = await fetch('https://bagbot-backend.onrender.com/upload-pdf', {
            method: 'POST',
            body: formData
        });
        const data = await res.json();
            if (!res.ok) {
                errorPDF = data.error || 'Error al procesar el PDF.';
                setTimeout(() => {errorPDF = ''; }, 5000);
                return;
            }
        filename = data.filename;
        showButtonDownload = true;

        } catch (e) {
        errorPDF = 'Error de conexión con el servidor.';
        setTimeout(() => {errorPDF = ''; }, 5000);
        }
        if (loggedIn){
            //Si esta logueado obtenemos la conversacion de la BD
            getChat_DB(false);
        }else{
            //Sino obtenemos la conversacion del JSON
            getChat_JSON();
        }
  }

async function downloadPDF() {
    try {
        const res = await fetch(`https://bagbot-backend.onrender.com/download-pdf?filename=${filename}`);
        if (!res.ok) {
            errorPDF = 'No se pudo descargar el resumen.';
            setTimeout(() => {errorPDF = ''; }, 5000);
            return;
        }
        const blob = await res.blob();
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `resumen_${filename}`;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
        showButtonDownload = false; //Deshabilita el boton Descargar
        optionsMenu('Ver Opciones'); //Mostrar Opciones
    } catch (e) {
        errorPDF = 'Error al descargar el archivo.';
        setTimeout(() => {errorPDF = ''; }, 5000);
    }
}
function optionsMenu(option) {
	selectedOption.set(option);
    showOptions.set(true);
    sendOptionSelected(option)
}
</script>

<div class="chatbox">
    <div class="messages" bind:this={chatEnd}>
        {#if chat.length === 0}
            {#each chatInit as message}
                <MessageBlock {message} on:optionSel={optionSelected} lastMessage=true />
            {/each}
        {:else}
            {#each chat as message, i (i)}
                <MessageBlock {message} {locked} on:optionSel={optionSelected} lastMessage={i === chat.length - 1} chatLength={chat.length}/>
            {/each}
        {/if}
    </div>
    <div class="input-container">
        {#if userOption === "📝 Resumir un recurso PDF" && !showButtonDownload}
            <div class="errorPDF {errorPDF.length > 0 ? '' : 'noShow'}"><span>{errorPDF}</span></div>
            <form class="input-container"  on:submit|preventDefault={uploadPDF}>
                <input class="input-box" type="file" name="file" accept="application/pdf" on:change={(e) => file = e.target.files[0]}/>
                <button class="btn btn-primary" type="submit">Subir y Resumir</button>
                <button class="btn btn-primary extra" on:click={() => optionsMenu("Ver Opciones")}>Ver Opciones</button>
            </form>
        {:else if userOption === "📝 Resumir un recurso PDF" && showButtonDownload}
            <button class="btn btn-primary" on:click={downloadPDF}>Descargar</button> 
        {:else}
            <input class="input-box" type="text" bind:value={userMessage} bind:this={inputRef} on:keydown={handleKeyPress} placeholder="Escribe tu mensaje..." disabled={showOpt || locked}/>
            <button class="btn btn-primary" on:click={sendMessage} disabled={showOpt || locked}>Enviar</button> 
        {/if}
    </div>
</div>


<style>
	.chatbox {
		box-shadow: 0 10px 20px rgba(0, 0, 0, 0.19), 0 6px 6px rgba(0, 0, 0, 0.23);
        padding: 20px 30px 12px;
		margin-top: 20px;
		width: 92%;
        height: calc(100vh - 328px) !important;
		display: flex;
		flex-direction: column;
		justify-content: end;
		background: #f7f7f7;
	}
    input {
		border: 1px solid transparent;
		padding: 8px 16px;
		margin-right: 16px;
		border-radius: 18px;
		box-shadow: 0 0 16px #00000029, 0rem 16px 16px -16px #00000029;
		font-weight: 400;
		font-size: 18px;
	}
    input:focus{
        outline: none;
        border: 1px solid #3980bb;
        box-shadow: 0 0 16px #2677c729, 0rem 16px 16px -16px #2677c729;
    }
	.messages {		
		width: 100%;
        overflow-y: auto;
        padding-right: 5px;
	}
    .messages::-webkit-scrollbar {
        width: 8px;  
    }

    .messages::-webkit-scrollbar-track {
        background: #edf4ff; 
    }

    .messages::-webkit-scrollbar-thumb {
        background-color: #3980bb;
        border-radius: 20px;
        border: 1px solid #eaf1ff;
    }
	.input-container {
		display: flex;
        margin-top: 15px;
        width: 100%;
        justify-content: center;
        position: relative;
	}
	.input-box {
		flex-grow: 1;
	}
    .errorPDF{
        position: absolute;
        color: red;
        background: #ffffff;
        padding: 12px 25px;
        border-radius: 8px;
        top: -22px;
        box-shadow: 0px 0px 8px #eeb8b8;
    }
    .errorPDF.noShow{
        display: none;
    }
    .btn.extra{
        margin-left: 10px;
    }
@media only screen and (max-width: 1280px) {
	input{
		font-size: 15px;
	}
}
</style>
