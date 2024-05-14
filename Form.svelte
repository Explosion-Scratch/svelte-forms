<script>
	import mime from 'mime-match';
	import { createEventDispatcher } from 'svelte';
	import { onMount } from 'svelte';
	import forwardEvents from './forwardevents.js';
	const dispatch = createEventDispatcher();
	const DEFAULT_INVALID_MESSAGE = 'Invalid';
	const INPUT_QS =
		'textarea, input:not([type=radio], [type=checkbox]), select, button, .checkboxes_container, .radio_container';
	const VALID_ATTR = 'data-valid';
	let GLOBALS = { val() {} };
	export let type = 'input',
		options = [],
		accept = '*/*',
		validate = ({ value, el }) => {
			if (typeof value === 'string') {
				return /^.+$/.test(value);
			} else if (el.matches('input[type=file]')) {
				if (Array.isArray(value) && value[0] instanceof File) {
					if (!value.every((file) => mime(file.type, accept))) {
						return `Filetype invalid (accepts ${accept})`;
					}
					return true;
				}
				return `You must upload a file`;
			} else if (Array.isArray(value)){
				return value?.length;
			} else if (typeof value === 'number'){
				return typeof parseFloat(value) === 'number'
			}
		},
		placeholder = '',
		description,
		id,
		data = {},
		container,
		label = '',
		errorMessage = '',
		value,
		min = 0,
		max = 100,
		step = 0.1;
	export const getVal = () => GLOBALS.val();

	let name = id;
	let VALID = true,
		ERROR_MESSAGE = '';
	onMount(() => {
		console.log(min, max)
		GLOBALS.val = val;
		value = $$props.default || value || '';
		type = type.toLowerCase();
		if (!id && type !== 'form') {
			throw new Error('No ID passed for input of type ' + type);
		}
		if (type !== 'form') {
			container.querySelector(INPUT_QS).id = id;
			forwardEvents(container.querySelector(INPUT_QS), { dispatch });
		} else {
			forwardEvents(container, { dispatch });
			setTimeout(val, 100);
		}
		container.addEventListener('input', handle);
		container.addEventListener('onkeyup', handle);
		container.addEventListener('change', handle);
		container.addEventListener('input', handle);
		container.addEventListener('blur', handle);
		container.addEventListener('run-validate', () => {
			if (type === 'form') {
				return;
			}
			handle({ target: container.querySelector(INPUT_QS), skipLabel: true });
		});
		async function val() {
			let els = [...container.querySelectorAll(INPUT_QS)];
			let e = new CustomEvent('run-validate');
			[...container.querySelectorAll('.container')].forEach((i) => i.dispatchEvent(e));
			await new Promise((r) => setTimeout(r));
			let out = Object.fromEntries(
				els
					.filter((i) => i.getAttribute(VALID_ATTR))
					.filter((i) => i.name || i.getAttribute('name'))
					.map((i) => [i.name || i.getAttribute('name'), getValue(i)])
			);
			dispatch('update', out);
			data = out;
			return out;
		}
		function handle(e) {
			let i;
			if (type === 'form') {
				i = e.target.closest(INPUT_QS);
			} else {
				i = container.querySelector(INPUT_QS);
				id = i.id;
			}

			let value = getValue(i);
			validate_this({ value });
			//TODO
			if (!i.name) {
					i.name = i.getAttribute('name');
				}
				if (i.getAttribute(VALID_ATTR)) {
					data[i.name] = value;
				} else {
					delete data[i.name];
				}
			if (type === 'form') {
				dispatch('update', data);
			}

			function validate_this({ value }) {
				ERROR_MESSAGE = undefined;
				let i = container.querySelector(INPUT_QS);
				if (!i) {
					return;
				}
				let { valid, message } = checkValidation({
					validate,
					el: i,
					id,
					value
				});
				let altMsg = '';
				if (i.matches('input[type=range]')){
					const rangeValid = i.valueAsNumber <= max && i.valueAsNumber >= min;
					if (!rangeValid){
						altMsg = `Value must be between ${min} and ${max}`
					}
					valid = valid && rangeValid;
				}
				if (!e.skipLabel) {
					VALID = valid;
					ERROR_MESSAGE = altMsg || message;
				}
				if (valid) {
					i.setAttribute(VALID_ATTR, 'true');
				} else {
					i.removeAttribute(VALID_ATTR);
				}
			}
		}
	});

	function checkValidation({ validate, el, id, value }) {
		if (!(el instanceof HTMLElement)) {
			return;
		}
		if (validate instanceof RegExp) {
			if (validate.test(value)) {
				return {
					valid: true
				};
			}
			return {
				valid: false
			};
		}

		if (validate instanceof Function) {
			let result = validate({ validate, el, id, value });
			if (result === true) {
				return {
					valid: true
				};
			}
			if (typeof result === 'string' || result === undefined) {
				return {
					valid: false,
					message: result
				};
			}
			return {
				valid: false
			};
		}
	}

	function getValue(input) {
		if (!(input instanceof HTMLElement)) {
			return;
		}
		if (input.matches('input[type=range]')){
			return input.valueAsNumber;
		}
		if (input.matches('.checkboxes_container')) {
			return [...input.querySelectorAll('input[type=checkbox]')]
				.filter((i) => i.checked)
				.map((i) => i.value);
		}
		if (input.matches('.radio_container')) {
			return [...input.querySelectorAll('input[type=radio]')].find((i) => i.checked).value;
		}
		if (input.matches('input[type=file]')) {
			return [...input.files];
		}
		return input?.value;
	}
</script>

<div
	id="s_form_{id || type}"
	class="container {label || description ? 'container_box' : ''}"
	bind:this={container}>
	{#if label}<label for={id} class="label {type}_label">{label}</label>{/if}
	{#if description}<label for={id} class="description {type}_description">{description}</label>{/if}
	{#if !VALID && type !== 'form'}
		<span class="error">{ERROR_MESSAGE || errorMessage || DEFAULT_INVALID_MESSAGE}</span>
	{/if}
	{#if type === 'input'}
		<input {placeholder} {name} {value} />
	{:else if type === 'number'}
		<input type='number' {placeholder} {name} {value} {step} {min} {max}>
	{:else if type === 'slider'}
		<div class='slider_container'>
			<div class='min'>{min}</div>
			<input {name} {value} type="range" min={min || 0} max={max || 10} step={step || "0.1"}/>
			<div class='max'>{max}</div>
			<div class='curr'>{data[name]}</div>
		</div>
	{:else if type === 'file'}
		<input type="file" {placeholder} {name} {accept} />
	{:else if type === 'radio'}
		<div class="radio_container" {name}>
			{#each options as option}
				<label
					><input
						type="radio"
						{name}
						value={option.value || option}
						checked={value === (option.value || option)} />{option.label || option}</label>
			{/each}
		</div>
	{:else if type === 'form'}
		<form on:submit|preventDefault={(e) => dispatch('submit', e)}>
			<slot />
		</form>
	{:else if type === 'checkboxes'}
		<div class="checkboxes_container" {name}>
			{#each options as option}
				<label
					><input
						type="checkbox"
						{name}
						value={option.value || option}
						checked={value.includes(options.value || option) || null} />{option.label ||
						option}</label>
			{/each}
		</div>
	{:else if type === 'select'}
		<select {id} {name}>
			{#each options as option}
				<option
					value={option.value || option}
					selected={value === option.value || value === option || null}
					>{option.label || option}</option>
			{/each}
		</select>
	{:else if type === 'textarea'}
		<textarea {name} {placeholder} {value} />
	{:else if type === 'button'}
		<button>
			<slot />
		</button>
	{/if}
</div>

<!--
<style>
.container#s_form_form {
    padding: 1rem;
    border: 1px solid;
    display: grid;
    
    .container {
      margin: .3rem;
      border: .2rem;
      display: flex;
      flex-direction: column;
      &:hover {
          background: #0001;
      }
      
      label.label {
          color: #777;
          font-size: 1.2rem;
      }
      
      .radio_container {
            border: 1px solid;
            border-radius: .3rem;
            padding: .5rem;
        }
    }
    
    textarea {
        height: 6rem;
    }
}
</style>
-->
<style>
	* {
		box-sizing: border-box;
		padding: 0;
		margin: 0;
	}
	.container#s_form_form {
		padding: 1rem;
		display: grid;
	}
	.container#s_form_form .container {
		margin: 0.3rem;
		border: 0.2rem;
		display: flex;
		flex-direction: column;
	}
	.container#s_form_form .container:hover {
		background: #00000005;
	}
	.container#s_form_form .container .label {
		font-size: 1.2rem;
		margin-bottom: 0.5rem;
		margin-top: 0.5rem;
		font-weight: 200;
	}
	.container#s_form_form .container.container_box {
		border: 1px solid #0001;
		border-radius: 0.3rem;
		padding: 0.5rem 1rem;
		position: relative;
	}
	.container#s_form_form textarea {
		height: 6rem;
	}

	input,
	select,
	textarea,
	button,
	::-webkit-file-upload-button {
		background: transparent;
		border: 1px solid #0001;
		padding: 0.5rem;
	}
	input[type='radio'] {
		margin-right: 0.4rem;
		/* Center stuff lol */
		transform: translateY(20%);
	}
	input[type='radio'],
	select,
	button,
	::-webkit-file-upload-button,
	input[type='button'],
	label {
		cursor: pointer;
	}
	:is(input, select, textarea, button, ::-webkit-file-upload-button):focus {
		outline: none;
		border: 1px solid #0002;
	}
	input[type='file'] {
		font-style: italic;
		color: #666;
	}
	.error {
		color: #933;
		margin: 0.1rem 0.1rem;
		background: #9001;
		padding: 0.3rem 0.5rem;
		border-radius: 0.2rem;
		border: 0.1rem solid #900;
		margin-bottom: 0.5rem;
	}
	.description {
		font-weight: 200;
		margin-bottom: 0.5rem;
		color: #666;
		font-size: 0.9rem;
	}
	.slider_container {
			display: flex;
			gap: 10px;
			align-items: center;
			position: relative;
	}
			.slider_container :is(.min, .max) {
				font-style: italic;
				font-size: .8em;
				color: #666666;
			}
			.slider_container .curr {
				font-weight: 600;
			}
			.slider_container .curr::before{content: '= '; margin-left: 20px;}
</style>
