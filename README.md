require('dotenv').config();
const fs = require('fs');
const {
  Client, GatewayIntentBits, EmbedBuilder, ActionRowBuilder, ButtonBuilder, ButtonStyle,
  ModalBuilder, TextInputBuilder, TextInputStyle, PermissionFlagsBits, ChannelType,
  SlashCommandBuilder, REST, Routes, AttachmentBuilder,
} = require('discord.js');

const env = process.env;
const list = (v) => (v || '').split(',').map((s) => s.trim()).filter(Boolean);

/* ------------------------------------------------------------------ */
/*  :hand_e6e8eb: Welcome [Username_Here] to Liberty County Roleplay!
To play in-game, verify your Roblox account: https://discord.gg/FpEGmWRj4⁠
Visit our sessions channel to jump right in with the community, we encourage you to check out our whitelisted information for additional insight on how our sessions operate*/
/* ------------------------------------------------------------------ */
const CONFIG = {
  serverName: 'Liberty County Roleplay',
  color: 0x1f5fbf,
  description:
    'Liberty County Roleplay is a realistic community roleplay server built around ' +
    'police, fire, EMS, dispatch, and civilian life. Join a department, work a job, ' +
    'or live your own story. We use a full CAD system with 911 calls, jobs, banking, and more.',
  departments: [
    'Georgia State Police',
    "Liberty County Sheriff's Office",
    'Fairview Police Department',
    'River City Police Department',
    'Liberty County Fire Department',
  ],
  gettingStarted: [
    'Read the server rules before you start.',
    'Pick up your roles and read the info channels.',
    'Open a ticket if you need help or want to join a department.',
  ],
};

const TYPES = {
  general: {
    label: 'General', emoji: '📩', style: ButtonStyle.Primary,
    desc: 'Questions, reports, and general help',
    staff: list(env.STAFF_GENERAL),
  },
  management: {
    label: 'Management', emoji: '🛡️', style: ButtonStyle.Secondary,
    desc: 'Department issues, appeals, and staff reports',
    staff: list(env.STAFF_MANAGEMENT),
  },
  ownership: {
    label: 'Ownership', emoji: '👑', style: ButtonStyle.Danger,
    desc: 'Serious concerns and partnerships',
    staff: list(env.STAFF_OWNERSHIP),
  },
};
/* ------------------------------------------------------------------ */

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMembers,   // needed for welcome messages
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent, // needed for ticket transcripts
  ],
});

/* ---------- tiny JSON store for the ticket counter ---------- */
const DB = 'data.json';
const loadDb = () => { try { return JSON.parse(fs.readFileSync(DB, 'utf8')); } catch { return { counter: 0 }; } };
const saveDb = (d) => fs.writeFileSync(DB, JSON.stringify(d));

/* ---------- slash commands ---------- */
const commands = [
  new SlashCommandBuilder()
    .setName('setup-tickets')
    .setDescription('Post the ticket panel in this channel')
    .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),
  new SlashCommandBuilder()
    .setName('welcome-preview')
    .setDescription('Preview the welcome message')
    .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),
  new SlashCommandBuilder()
    .setName('ticket')
    .setDescription('Ticket tools')
    .addSubcommand((s) => s.setName('add').setDescription('Add a user to this ticket')
      .addUserOption((o) => o.setName('user').setDescription('User to add').setRequired(true)))
    .addSubcommand((s) => s.setName('remove').setDescription('Remove a user from this ticket')
      .addUserOption((o) => o.setName('user').setDescription('User to remove').setRequired(true)))
    .addSubcommand((s) => s.setName('close').setDescription('Close this ticket')),
].map((c) => c.toJSON());

client.once('clientReady', async () => {
  try {
    const rest = new REST({ version: '10' }).setToken(env.TOKEN);
    await rest.put(Routes.applicationGuildCommands(client.user.id, env.GUILD_ID), { body: commands });
    console.log(`Logged in as ${client.user.tag} and registered commands.`);
  } catch (e) {
    console.error('Failed to register commands:', e);
  }
});

/* ---------- welcome ---------- */
function welcomeEmbed(member) {
  return new EmbedBuilder()
    .setColor(CONFIG.color)
    .setTitle(`Welcome to ${CONFIG.serverName}!`)
    .setDescription(CONFIG.description)
    .setThumbnail(member.user.displayAvatarURL())
    .addFields(
      { name: 'Departments', value: CONFIG.departments.map((d) => `• ${d}`).join('\n') },
      { name: 'Getting started', value: CONFIG.gettingStarted.map((g, i) => `${i + 1}. ${g}`).join('\n') },
    )
    .setFooter({ text: `You are member #${member.guild.memberCount}` })
    .setTimestamp();
}

client.on('guildMemberAdd', async (member) => {
  try {
    if (env.AUTO_ROLE_ID) await member.roles.add(env.AUTO_ROLE_ID).catch(() => {});
    const ch = await member.guild.channels.fetch(env.WELCOME_CHANNEL_ID).catch(() => null);
    if (!ch) return;
    await ch.send({ content: `Welcome ${member}!`, embeds: [welcomeEmbed(member)] });
  } catch (e) {
    console.error('Welcome error:', e);
  }
});

/* ---------- ticket helpers ---------- */
const parseTopic = (t) => {
  const m = /^ticket:(\d+):(\w+)$/.exec(t || '');
  return m ? { userId: m[1], type: m[2] } : null;
};
const isStaff = (member, type) =>
  member.permissions.has(PermissionFlagsBits.Administrator) ||
  member.roles.cache.some((r) => (TYPES[type]?.staff || []).includes(r.id));

function panelMessage() {
  const embed = new EmbedBuilder()
    .setColor(CONFIG.color)
    .setTitle(`${CONFIG.serverName} Support`)
    .setDescription(
      'Need help? Choose the ticket type that fits your inquiry.\n\n' +
      Object.values(TYPES).map((t) => `${t.emoji} **${t.label}** - ${t.desc}`).join('\n'),
    );
  const row = new ActionRowBuilder().addComponents(
    ...Object.entries(TYPES).map(([key, t]) =>
      new ButtonBuilder().setCustomId(`ticket:open:${key}`).setLabel(t.label).setEmoji(t.emoji).setStyle(t.style)),
  );
  return { embeds: [embed], components: [row] };
}

function ticketModal(type) {
  return new ModalBuilder()
    .setCustomId(`ticket:modal:${type}`)
    .setTitle(`${TYPES[type].label} ticket`)
    .addComponents(
      new ActionRowBuilder().addComponents(
        new TextInputBuilder().setCustomId('subject').setLabel('Subject')
          .setStyle(TextInputStyle.Short).setMaxLength(80).setRequired(true)),
      new ActionRowBuilder().addComponents(
        new TextInputBuilder().setCustomId('details').setLabel('Describe your inquiry')
          .setStyle(TextInputStyle.Paragraph).setMaxLength(1500).setRequired(true)),
    );
}

async function createTicket(i, type) {
  await i.deferReply({ ephemeral: true });
  const existing = i.guild.channels.cache.find((c) => c.topic && c.topic.startsWith(`ticket:${i.user.id}:`));
  if (existing) return i.editReply(`You already have an open ticket: ${existing}`);

  const db = loadDb();
  db.counter += 1;
  saveDb(db);
  const num = String(db.counter).padStart(4, '0');
  const subject = i.fields.getTextInputValue('subject');
  const details = i.fields.getTextInputValue('details');
  const staffRoles = TYPES[type].staff;
  const allow = [
    PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages,
    PermissionFlagsBits.ReadMessageHistory, PermissionFlagsBits.AttachFiles,
  ];

  const channel = await i.guild.channels.create({
    name: `${type}-${num}`,
    type: ChannelType.GuildText,
    parent: env.TICKET_CATEGORY_ID || null,
    topic: `ticket:${i.user.id}:${type}`,
    permissionOverwrites: [
      { id: i.guild.id, deny: [PermissionFlagsBits.ViewChannel] },
      { id: i.user.id, allow },
      ...staffRoles.map((id) => ({ id, allow })),
      { id: client.user.id, allow: [...allow, PermissionFlagsBits.ManageChannels] },
    ],
  });

  const embed = new EmbedBuilder()
    .setColor(CONFIG.color)
    .setTitle(`${TYPES[type].emoji} Ticket #${num} - ${subject}`)
    .setDescription(details)
    .addFields(
      { name: 'Opened by', value: `${i.user}`, inline: true },
      { name: 'Type', value: TYPES[type].label, inline: true },
      { name: 'Claimed by', value: 'Nobody yet', inline: true },
    )
    .setTimestamp();
  const row = new ActionRowBuilder().addComponents(
    new ButtonBuilder().setCustomId('ticket:claim').setLabel('Claim').setStyle(ButtonStyle.Success),
    new ButtonBuilder().setCustomId('ticket:close').setLabel('Close').setStyle(ButtonStyle.Danger),
  );
  const ping = staffRoles.map((id) => `<@&${id}>`).join(' ');
  await channel.send({ content: `${i.user} ${ping}`.trim(), embeds: [embed], components: [row] });
  return i.editReply(`Your ticket has been created: ${channel}`);
}

async function transcript(channel) {
  const all = [];
  let last;
  for (;;) {
    const batch = await channel.messages.fetch({ limit: 100, before: last });
    if (!batch.size) break;
    all.push(...batch.values());
    last = batch.last().id;
    if (all.length >= 1000) break;
  }
  return all.reverse().map((m) => {
    const files = m.attachments.size ? ' ' + [...m.attachments.values()].map((a) => a.url).join(' ') : '';
    return `[${m.createdAt.toISOString()}] ${m.author.tag}: ${m.content || ''}${files}`;
  }).join('\n') || '(no messages)';
}

async function closeTicket(i, reason) {
  const info = parseTopic(i.channel.topic);
  if (!info) return i.reply({ content: 'This is not a ticket channel.', ephemeral: true });
  if (i.user.id !== info.userId && !isStaff(i.member, info.type)) {
    return i.reply({ content: 'Only the ticket owner or staff can close this ticket.', ephemeral: true });
  }
  await i.reply('Closing this ticket in 5 seconds...');
  const text = await transcript(i.channel);
  const name = `${i.channel.name}.txt`;
  const embed = new EmbedBuilder()
    .setColor(CONFIG.color)
    .setTitle(`Ticket closed: ${i.channel.name}`)
    .addFields(
      { name: 'Opened by', value: `<@${info.userId}>`, inline: true },
      { name: 'Closed by', value: `${i.user}`, inline: true },
      { name: 'Reason', value: reason || 'No reason given' },
    )
    .setTimestamp();

  const log = env.TRANSCRIPT_CHANNEL_ID && await i.guild.channels.fetch(env.TRANSCRIPT_CHANNEL_ID).catch(() => null);
  if (log) await log.send({ embeds: [embed], files: [new AttachmentBuilder(Buffer.from(text), { name })] });
  const user = await client.users.fetch(info.userId).catch(() => null);
  if (user) await user.send({ embeds: [embed], files: [new AttachmentBuilder(Buffer.from(text), { name })] }).catch(() => {});
  setTimeout(() => i.channel.delete().catch(() => {}), 5000);
}

/* ---------- interactions ---------- */
client.on('interactionCreate', async (i) => {
  try {
    if (i.isChatInputCommand()) {
      if (i.commandName === 'setup-tickets') {
        await i.channel.send(panelMessage());
        return i.reply({ content: 'Ticket panel posted.', ephemeral: true });
      }
      if (i.commandName === 'welcome-preview') {
        return i.reply({ embeds: [welcomeEmbed(i.member)], ephemeral: true });
      }
      if (i.commandName === 'ticket') {
        const info = parseTopic(i.channel.topic);
        if (!info) return i.reply({ content: 'Use this inside a ticket channel.', ephemeral: true });
        const sub = i.options.getSubcommand();
        if (sub === 'close') {
          return i.showModal(new ModalBuilder().setCustomId('ticket:closemodal').setTitle('Close ticket').addComponents(
            new ActionRowBuilder().addComponents(
              new TextInputBuilder().setCustomId('reason').setLabel('Reason (optional)')
                .setStyle(TextInputStyle.Short).setRequired(false)),
          ));
        }
        if (!isStaff(i.member, info.type)) return i.reply({ content: 'Staff only.', ephemeral: true });
        const user = i.options.getUser('user');
        if (sub === 'add') {
          await i.channel.permissionOverwrites.edit(user.id, { ViewChannel: true, SendMessages: true, ReadMessageHistory: true });
          return i.reply(`${user} was added to the ticket.`);
        }
        if (sub === 'remove') {
          await i.channel.permissionOverwrites.delete(user.id);
          return i.reply(`${user} was removed from the ticket.`);
        }
      }
    } else if (i.isButton()) {
      const [ns, action, type] = i.customId.split(':');
      if (ns !== 'ticket') return;
      if (action === 'open') return i.showModal(ticketModal(type));
      const info = parseTopic(i.channel.topic);
      if (!info) return;
      if (action === 'claim') {
        if (!isStaff(i.member, info.type)) return i.reply({ content: 'Only staff can claim tickets.', ephemeral: true });
        const msg = i.message;
        const embed = EmbedBuilder.from(msg.embeds[0]);
        const fields = embed.data.fields.map((f) => (f.name === 'Claimed by' ? { ...f, value: `${i.user}` } : f));
        embed.setFields(fields);
        await msg.edit({ embeds: [embed] });
        return i.reply(`${i.user} has claimed this ticket.`);
      }
      if (action === 'close') {
        return i.showModal(new ModalBuilder().setCustomId('ticket:closemodal').setTitle('Close ticket').addComponents(
          new ActionRowBuilder().addComponents(
            new TextInputBuilder().setCustomId('reason').setLabel('Reason (optional)')
              .setStyle(TextInputStyle.Short).setRequired(false)),
        ));
      }
    } else if (i.isModalSubmit()) {
      const [ns, action, type] = i.customId.split(':');
      if (ns !== 'ticket') return;
      if (action === 'modal') return createTicket(i, type);
      if (action === 'closemodal') return closeTicket(i, i.fields.getTextInputValue('reason'));
    }
  } catch (err) {
    console.error(err);
    const m = { content: 'Something went wrong. Please try again.', ephemeral: true };
    if (i.isRepliable()) (i.deferred || i.replied ? i.followUp(m) : i.reply(m)).catch(() => {});
  }
});

client.login(env.TOKEN);
