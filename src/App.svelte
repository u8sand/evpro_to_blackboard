<script>
import blobToStream from 'blob-to-stream'
import parseRTF from 'rtf-parser'
import iconv from 'iconv-lite'
import encodings from 'iconv-lite/encodings'
iconv.encodings = {...encodings, symbol: 'windows1254'}

let files
let internal = {}
let inputContent
let internalContent
let outputContent
let visibleInternals = false

function match_all(re, str) {
  const results = []
  let m
  while (m = re.exec(str)) {
    results.push(m.groups)
  }
  if (results.length === 0) {
    console.warn(`${re} of ${str} was empty`)
  }
  return results
}

function decode_true_false(true_false) {
  const qa = (
    /\n\d+\.\s+(?<question>.+)\n+ANS:\s+(?<answer>[TF])/gm
  )
  return match_all(qa, true_false)
}

function encode_true_false(true_false) {
  const answer_lookup = {
    'T': 'True',
    'F': 'False',
  }
  const results = []
  for (const { question, answer } of true_false) {
    results.push(
      `TF\t${question.replace(/\t+/g, ' ')}\t${answer_lookup[answer]}\tincorrect`
    )
  }
  return results.join('\n\n')
}

function decode_multiple_choice(multiple_choice) {
  const mc = (
    /\n\d+\.\s+(?<question>.+)\n+(?<choices>([a-z]\.\s+.+?[\t\n])+)\n+ANS:\s+(?<answer>[A-Z])/gm
  )
  const decoded = []
  for (const { question, choices, answer } of match_all(mc, multiple_choice)) {
    const c = (
      /(?<label>[a-z])\.\s+(?<choice>.+?)[\t\n]/gm
    )

    const decoded_choices = []
    for (const { label, choice } of match_all(c, choices)) {
      decoded_choices.push({
        label,
        choice: choice.replace(/\t+/g, ' '),
      })
    }

    decoded.push({
      question: question.replace(/\t+/g, ' '),
      choices: decoded_choices,
      answer,
    })
  }
  return decoded
}

function encode_multiple_choice(multiple_choice) {
  const results = []
  for (const { question, choices, answer} of multiple_choice) {
    const answer_results = []
    for (const { label, choice } of choices) {
      answer_results.push(
        `${choice}\t${label.toUpperCase() === answer.toUpperCase() ? 'correct' : 'incorrect'}`
      )
    }
    results.push(`MC\t${question}\t${answer_results.join('\t')}`)
  }
  return results.join('\n\n')
}

function decode_matching(matching) {
  const m = (
    /(?<question>.+)?\n(?<answers>a\.\s+.+?[\t\n]([a-z]\.\s+.+?[\t\n])+)\n+((?<questions>((\d+\.\s+.+)\n+)+?))((?<question_answers>((\d+\.\s+ANS:\s+[A-Z])\n{1,2})+))/gm
  )
  const decoded = []
  for (const { question, answers, questions, question_answers } of match_all(m, matching)) {
    const a = (
      /(?<label>[a-z])\.\s+(?<answer>.+?)[\t\n]/gm
    )
    const q = (
      /(?<label>\d+)\.\s+(?<question>.+)\n/gm
    )
    const qa = (
      /(?<question_label>\d+)\.\s+ANS:\s+(?<answer_label>.+)\n/gm
    )

    const decoded_answers = []
    for (const { label, answer } of match_all(a, answers)) {
      decoded_answers.push({
        label,
        answer: answer.replace(/\t+/g, ' '),
      })
    }
    const decoded_questions = []
    for (const { label, question } of match_all(q, questions)) {
      decoded_questions.push({
        label,
        question: question.replace(/\t+/g, ' '),
      })
    }
    const decoded_question_answers = []
    for (const { question_label, answer_label } of match_all(qa, question_answers)) {
      decoded_question_answers.push({
        question_label,
        answer_label,
      })
    }

    decoded.push({
      question: (question || 'Unnamed').replace(/\t+/g, ' '),
      answers: decoded_answers,
      questions: decoded_questions,
      question_answers: decoded_question_answers,
    })
  }
  return decoded
}

function encode_matching(matching) {
  const results = []
  for (const { question, answers, questions, question_answers } of matching) {
    const question_lookup = {}
    for (const { label, question } of questions) {
      // 1., 2., 3. => question
      question_lookup[label.toUpperCase()] = question
    }
    
    const answer_question_lookup = {}
    for (const { question_label, answer_label } of question_answers) {
      // 1., 2., 3. => A., B., C.
      answer_question_lookup[answer_label.toUpperCase()] = question_label
    }

    const question_answer_results = []
    for (const { label: answer_label, answer } of answers) {
      const question_label = answer_question_lookup[answer_label.toUpperCase()]
      const question = question_label === undefined ? 'Unassigned' : question_lookup[question_label]
      question_answer_results.push(
        `${question}\t${answer}`
      )
    }
    results.push(
      `MAT\t${question}\t${question_answer_results.join('\t')}`
    )
  }
  return results.join('\n\n')
}

function decode_document(text) {
  const document = `${text.replace(/\n([A-Z\/ ]+)\n/gm, '\nSECTION: $1\n')}\nEND OF FILE`
  const section = (
    /\nSECTION: (?<section>.+)\n(?<content>(.|\n)+?)(?=(\nSECTION:)|END OF FILE)/gm
  )
  return {
    title: document.substring(0, document.indexOf('\n')),
    sections: match_all(section, document)
  }
}

function encode_document({ title, sections }) {
  const results = []
  internal = { title, sections }
  for (const { section, content } of sections) {
    if (section === 'TRUE/FALSE') {
      const decoded = decode_true_false(content)
      internal[section] = decoded
      const encoded = encode_true_false(decoded)
      results.push(encoded)
    } else if (section === 'MULTIPLE CHOICE') {
      const decoded = decode_multiple_choice(content)
      internal[section] = decoded
      const encoded = encode_multiple_choice(decoded)
      results.push(encoded)
    } else if (section === 'MATCHING') {
      const decoded = decode_matching(content)
      internal[section] = decoded
      const encoded = encode_matching(decoded)
      results.push(encoded)
    } else {
      console.warn(`Ignoring section: ${section}`)
    }
  }
  internalContent = JSON.stringify(internal)
  return results.join('\n\n')
}

function submit() {
  try {
    const decoded_document = decode_document(inputContent)
    outputContent = encode_document(decoded_document)
  } catch (e) {
    console.error(e)
    outputContent = `Error while parsing`
  }
}

function download() {
  const element = document.createElement('a');
  element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(outputContent));
  element.setAttribute('download', internal.title !== undefined ? `${internal.title.replace(/\s+/, '_')}.txt` : 'output.txt');

  element.style.display = 'none';
  document.body.appendChild(element);

  element.click();

  document.body.removeChild(element);
}

async function updateContent() {
  try {
    parseRTF.stream(blobToStream(files[0]), async (err, doc) => {
      try {
        if (err) { throw err }
        inputContent = doc.content.map(({ content }) => content.map(({ value }) => value.trim()).join('\t').trim()).join('\n')
        submit()
      } catch (e) {
        console.error(e)
        outputContent = 'ERROR Parsing Document. Is it .rtf?'
      }
    })
  } catch (e) {
    console.error(e)
    outputContent = 'ERROR Reading Document.'
  }
}

function toggleInternal() {
  visibleInternals = !visibleInternals
}

</script>

<style>
  textarea {
    width: 100%;
    height: 200px;
  }
</style>

<p>
  <input type="file" bind:files on:change={updateContent}>
</p>

<p>
  <button on:click={toggleInternal}>Toggle Internals</button>
</p>

{#if visibleInternals}
  <fieldset>
    <legend>Input</legend>
    <textarea bind:value={inputContent}></textarea>
  </fieldset>

  <p>
    <button on:click={submit}>Submit</button>
  </p>

  <fieldset>
    <legend>Internal</legend>
    <textarea bind:value={internalContent}></textarea>
  </fieldset>
{/if}

<fieldset>
  <legend>Output</legend>
  {#if internal.title}
    <p><b>{internal.title}</b></p>
  {/if}
  <textarea bind:value={outputContent}></textarea>
</fieldset>

<p>
  <button on:click={download}>Download</button>
</p>
