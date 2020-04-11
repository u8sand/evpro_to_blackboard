<script>
let inputContent
let internalContent
let outputContent

function match_all(re, str) {
  const results = []
  let m
  while (m = re.exec(str)) {
    results.push(m.groups)
  }
  return results
}

function decode_document(text) {
  const document = (
    /^(?<title>.+)?\n+TRUE\/FALSE\n+(?<true_false>((.+)\n+)+)\n+MULTIPLE CHOICE\n+(?<multiple_choice>((.+)\n+)+)MATCHING\n+(?<matching>((.+)\n+)+)$/gm
  ).exec(inputContent)
  return document.groups
}

function decode_true_false(true_false) {
  const qa = (
    /\t\d+\.\t(?<question>.+)\n+ANS:\t(?<answer>[TF])/gm
  )
  return match_all(qa, true_false)
}

function encode_true_false(true_false) {
  const answer_lookup = {
    'T': 'True',
    'F': 'False',
  }
  const results = []
  for (const tf of true_false) {
    results.push(
      `TF\t${tf.question}\t${answer_lookup[tf.answer]}\tincorrect`
    )
  }
  return results.join('\n\n')
}

function decode_multiple_choice(multiple_choice) {
  const mc = (
    /\t\d+\.\t(?<question>.+)\n(?<choices>([a-z]\.\n.+\n)+)\n+ANS:\t(?<answer>[A-Z])/gm
  )
  const decoded = []
  for (const { question, choices, answer } of match_all(mc, multiple_choice)) {
    const c = (
      /(?<label>[a-z])\.\n(?<choice>.+)\n/gm
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
    /(?<question>.+)?\n(?<answers>a\.\n.+\n([a-z]\.\n.+\n)+)\n+((?<questions>((\t\d+\.\t.+)\n+)+?))((?<question_answers>((\t\d+\.\tANS:\t[A-Z])\n{1,2})+))/gm
  )
  const decoded = []
  for (const { question, answers, questions, question_answers } of match_all(m, matching)) {
    const a = (
      /(?<label>[a-z])\.\n(?<answer>.+)\n/gm
    )
    const q = (
      /\t(?<label>\d+)\.\t(?<question>.+)\n/gm
    )
    const qa = (
      /\t(?<question_label>\d+)\.\tANS:\t(?<answer_label>.+)\n+/gm
    )
    console.log(question_answers)

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
      question: (question || '').replace(/\t+/g, ' '),
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
      const question = question_label === undefined ? '' : question_lookup[question_label]
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

function submit() {
  try {
    const {
      title,
      true_false,
      multiple_choice,
      matching,
    } = decode_document(inputContent)
    const true_false_decoded = decode_true_false(true_false)
    const multiple_choice_decoded = decode_multiple_choice(multiple_choice)
    const matching_decoded = decode_matching(matching)

    const true_false_encoded = encode_true_false(true_false_decoded)
    const multiple_choice_encoded = encode_multiple_choice(multiple_choice_decoded)
    const matching_encoded = encode_matching(matching_decoded)

    // internalContent = JSON.stringify({
    //   title,
    //   true_false,
    //   multiple_choice,
    //   matching,
    //   true_false_decoded,
    //   multiple_choice_decoded,
    //   matching_decoded,
    //   true_false_encoded,
    //   multiple_choice_encoded,
    //   matching_encoded,
    // })

    outputContent = `${true_false_encoded}\n\n${multiple_choice_encoded}\n\n${matching_encoded}`
  } catch (e) {
    outputContent = `Error while parsing`
  }
}

</script>

<style>
  textarea {
    width: 100%;
    height: 200px;
  }
</style>

<fieldset>
  <legend>Input</legend>
  <textarea bind:value={inputContent}></textarea>
</fieldset>

<center>
  <button on:click={submit}>Submit</button>
</center>

<!-- 
<fieldset>
  <legend>Internal</legend>
  <textarea bind:value={internalContent}></textarea>
</fieldset>
 -->
 
<fieldset>
  <legend>Output</legend>
  <textarea bind:value={outputContent}></textarea>
</fieldset>
