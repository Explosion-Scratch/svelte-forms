<script>
	import Form from "./Form.svelte";
	//Bound data from the form. Only valid results are shown
	let data = {};
</script>

<!-- TODO: Add plugin support for things like code editor, custom selects, color picker, etc -->

<!-- Bind data, listen for updates, see more stuff in Form.svelte -->
<Form type="form" bind:data={data} on:update={(a) => console.log("Updated: ", a.detail)}>
  <!-- Radio input	 -->
	<Form type='radio' id="select" options={["one", "two", "three", "four"]} value="two" label="What's your favorite number?" description="I understand these are not actual colors just pick one."/>
	<Form type='select' id="select" options={["no", "still no", "fine", "yes", "YES YOURE AMAZING", "okokok ill star it on github"]} value="fine" label="Do you like this forms library?" description="Cool select menus"/>
	<!-- Normal input input	 -->
	<Form type="input" placeholder="Placeholder!!!" value="testing" id="input" label="This is a cool label"/>
	<!-- Textarea	 -->
	<Form type="textarea" placeholder="This is a textarea" id="textarea" description="Get some lorem ipsum and paste it here" value="Default" label="Lorem Ipsum"/>
	<!-- Custom validation function or regex, can return a string (for error), or a boolean	 -->
	<Form type="input" placeholder="Value must me 'secret'" validate={({value}) => value === "secret" ? true : `Invalid: ${value}`} id="validation"/>
		<!-- Button – No value	 -->
		<Form type="button" id="btn" on:click={() => alert("Clicked")}>Hello world</Form>
		<!-- 	Checkboxes!	 -->
		<Form type="checkboxes" id="checkboxes" options={["one", "two", "three"]} on:click={() => console.log("Clicked")} value={["two", "three"]}></Form>
</Form>
	
<!-- Result	 -->
<textarea value={JSON.stringify(data, null, 2)}></textarea>
	
	<style>
		:global(body){
			display: flex;
			justify-content: center;
			align-items: center;
			min-height: 100vh;
			height: fit-content;
			overflow-y: scroll;
		}
		textarea {
			font-family: monospace;
			height: 100vh;
			align-self: flex-start;
			margin-top: 1rem;
			width: 30vw;
			border: 1px solid #0001;
			border-radius: .2rem;
		}
	</style>
